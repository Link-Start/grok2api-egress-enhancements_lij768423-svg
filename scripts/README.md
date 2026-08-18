# scripts

| 文件 | 作用 |
| --- | --- |
| `from_residential.py` | 家宽 dump → Mihomo + Grok2API 节点表 + Guard 默认 |
| `from_residential_test.py` | 单测 |

```bash
python3 scripts/from_residential.py residential.dump --out-dir ~/grok-stack/egress-gen
```

AI 规范：[docs/AI_GROK2API_INSTALL.md](../docs/AI_GROK2API_INSTALL.md)
