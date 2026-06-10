# ether-claude-plugins

이더컴퍼니 사내 운영 자동화를 위한 **Claude Code 플러그인 마켓플레이스**입니다.

## 설치 방법

Claude Code 또는 Cowork에서 마켓플레이스를 추가:

```
/plugin marketplace add either-cmyk/claude-plugins
```

그 다음 원하는 플러그인을 설치:

```
/plugin install n-value-calculator@ether-claude-plugins
```

## 포함된 플러그인

| 플러그인 | 설명 | 주 트리거 |
|---|---|---|
| [n-value-calculator](./plugins/n-value-calculator) | CN인사이더 발주의 1위안당 원화(N값)를 자동 계산하여 수입원가 구글시트에 입력 | `1위안당금액 계산해줘` / `위안당원화 계산해줘` |

## 구조

```
claude-plugins/
├── .claude-plugin/
│   └── marketplace.json       # 마켓플레이스 매니페스트
├── plugins/
│   └── n-value-calculator/
│       ├── .claude-plugin/
│       │   └── plugin.json    # 플러그인 매니페스트
│       ├── skills/
│       │   └── n-value-calculator/
│       │       └── SKILL.md   # 스킬 본문
│       └── README.md
├── README.md
└── .gitignore
```

## 라이선스

MIT

