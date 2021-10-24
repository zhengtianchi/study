# MeshServer数据库约束

## MySQL版本

8.0.12

## 公共部分

gorm.Model

| 字段名     | 类型      | 必要 | 字段描述 |      |
| ---------- | --------- | ---- | -------- | ---- |
| id         | integer   | true | id       | 主键 |
| created_at | time.Time | true | 创建时间 |      |
| updated_at | time.Time | true | 修改时间 |      |
| deleted_at | DeletedAt | true | 删除时间 | 索引 |

