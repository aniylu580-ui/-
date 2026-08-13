# 数据模型（Data Model）

## 0. 全局约定

- 金额：一律 `amount_cents INTEGER`（分）；禁止浮点
- 时间：`timestamptz`（UTC 存储）；API 用 ISO 8601；展示按本地时区
- 主键：自增 BIGINT `id`
- 软删除：`is_deleted BOOLEAN DEFAULT FALSE`（交易、类目、记事谱）
- 审计：关键操作写 `audit_logs`
- 所有表含 `created_at`、`updated_at`；外键列建索引

## 1. 实体

### users（v1 预留，单用户）

- id, username UNIQUE, password_hash, created_at, updated_at
- 说明：v1 只允许一个用户，但所有业务表都带 `user_id`，为多用户留扩展

### accounts（账户/账本）

- id, user_id, name
- type ENUM(cash, wechat, alipay, bank_card, other)
- currency VARCHAR(3) DEFAULT 'CNY'
- is_deleted, created_at, updated_at

### categories（类目）

- id, user_id, name, parent_id（可空，预留二级类目）, icon, color
- is_system BOOLEAN（默认类目不可删）
- is_deleted, created_at, updated_at

### merchants（商户）

- id, user_id, name_normalized VARCHAR
- display_name（原始展示名）
- category_id（默认分类，可空）
- UNIQUE(user_id, name_normalized)
- created_at, updated_at

说明：`name_normalized` 为归一化后的商户名（全半角/空格/常见后缀），作为匹配键。

### transactions（交易）

- id, user_id, account_id, import_batch_id（可空 = 手动记录）
- platform ENUM(wechat, alipay, manual)
- occurred_at timestamptz（交易发生时间）
- recorded_at timestamptz（记录时间，即导入或手动记录的时间）
- raw_payee（原始对方名）, merchant_id（可空）
- amount_cents INTEGER > 0
- direction ENUM(income, expense)
- txn_type ENUM(consume, transfer, refund, repay, receive, aa_split, other)
- category_id（可空，待分类）
- classification_source ENUM(rule, merchant_memory, ai, manual, unclassified)
- classification_confidence NUMERIC(3,2)（0–1，可空）
- status ENUM(pending, confirmed, corrected)
- note TEXT
- 去重键：UNIQUE(user_id, platform, platform_txn_id, merchant_txn_id, amount_cents, occurred_at)；仅对导入交易（platform != manual）启用
- is_deleted, created_at, updated_at

### import_batches（导入批次）

- id, user_id, account_id, platform ENUM(wechat, alipay)
- file_name, file_hash VARCHAR(64)（SHA-256，防重复导入）
- file_size_bytes
- status ENUM(uploaded, parsing, parsed, failed)
- stats JSONB（总行数/成功/失败/去重数）
- imported_at, created_at, updated_at

### merchant_rules（记事谱）

- id, user_id, merchant_name_normalized
- category_id
- source ENUM(user_confirmed, ai_suggested)（用户确认的优先级最高）
- match_count INTEGER DEFAULT 0
- last_matched_at, is_deleted
- UNIQUE(user_id, merchant_name_normalized)
- created_at, updated_at

说明：`merchant_rules` 是"商户名 → 类目"的记忆表，独立于 `merchants`（后者是解析出的实体，前者是用户确认过的规则）。

### match_groups（对账组）

- id, user_id
- match_type ENUM(refund, repay_same_day, aa_split)
- status ENUM(proposed, confirmed, rejected)
- net_amount_cents INTEGER（AA 分账的净支出；退款/当天还为 0）
- reason TEXT（系统匹配理由）
- confirmed_at, created_at, updated_at

### match_group_items（对账组成员）

- id, match_group_id, transaction_id
- role ENUM(original, counterpart)

说明：一组至少 2 条；退款/当天还为一对支出+收入；AA 分账为一条支出 + 一条收入。

### audit_logs（审计日志）

- id, user_id, action, entity_type, entity_id, detail JSONB, created_at
- 关键动作：导入、删除交易、修改分类、确认/撤销对账组、修改记事谱

## 2. 关系要点

- transaction → account（多对一）；transaction → category（多对一，可空）
- transaction → merchant（多对一，可空）；merchant → category（默认分类）
- merchant_rules 独立于 merchants，两者都通过 `name_normalized` 关联到交易
- match_group ↔ transaction 通过 match_group_items
- 导入批次 → 交易（一对多）

## 3. 去重与幂等

- 导入去重键：platform + platform_txn_id + merchant_txn_id + amount_cents + occurred_at
- 同一文件再次导入：`file_hash` 命中即拒绝（返回"已导入过"）
- 手动记录不参与导入去重

## 4. 枚举值（v1）

- direction：income / expense
- txn_type：consume / transfer / refund / repay / receive / aa_split / other
- classification_source：rule / merchant_memory / ai / manual / unclassified
- match_type：refund / repay_same_day / aa_split
- match status：proposed / confirmed / rejected
- platform：wechat / alipay / manual

## 5. 后续扩展（已预留，不实现）

- 多用户（`user_id` 已存在）
- 多币种（`accounts.currency`；金额仍以分存，扩展时加汇率字段）
- 预算（budgets 表，按类目/周期）
