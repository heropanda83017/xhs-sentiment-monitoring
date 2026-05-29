# Cronjob Skill 名称解析坑点

## 问题描述
cronjob 中 `skills: ["real-estate-marketing"]` 引用的是 **分类名** 而非 **技能名**。

## 正确做法
- **技能名** = `xhs-sentiment-monitoring`（skill_manage 创建时的 name 参数）
- **分类名** = `real-estate-marketing`（skill_manage 创建时的 category 参数）
- cronjob 的 `skills` 字段必须用 **技能名**，分类名不会自动解析到其下的技能

## 举例
```
❌ skills: ["real-estate-marketing"]        # 分类名，不匹配任何技能
✅ skills: ["xhs-sentiment-monitoring"]      # 技能名，正确加载
```

## 修复方式
```python
cronjob(action='update', job_id='<id>', skills=['xhs-sentiment-monitoring'])
```

## 受影响的历史job
- `806ebce77bad` (macro-weekly-snapshot) — 已修复 ✅
- `7645a83f23b8` (macro-monthly-deep-report) — 已修复 ✅
