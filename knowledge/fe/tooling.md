# Tooling — 개발 도구 설정

> 좋은 도구는 좋은 코드를 강제한다. 자동화로 일관성을 확보하라.

## 핵심 원칙
- **자동화**: 린팅, 포매팅, 타입 검사는 수동이 아닌 자동
- **일관성**: 팀 전체 동일한 도구 버전, 동일한 설정
- **빠른 피드백**: 저장 시 즉시 에러 표시
- **에디터 독립성**: VS Code 설정은 권장사항, 강제하지 않음

## 필수 도구 스택
| 도구 | 역할 | 설정 파일 |
|------|------|-----------|
| TypeScript | 타입 검사 | tsconfig.json |
| ESLint | 코드 린팅 | eslint.config.js |
| Prettier | 코드 포매팅 | .prettierrc |
| Husky | git hooks | .husky/ |
| lint-staged | 커밋 전 검사 | package.json |

## 코드 예시

✅ tsconfig.json (엄격 설정)
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

✅ ESLint 설정 (Next.js + TypeScript)
```javascript
// eslint.config.js
import nextPlugin from '@next/eslint-plugin-next'
import tsPlugin from '@typescript-eslint/eslint-plugin'
import tsParser from '@typescript-eslint/parser'

export default [
  {
    files: ['**/*.ts', '**/*.tsx'],
    plugins: {
      '@typescript-eslint': tsPlugin,
      '@next/next': nextPlugin,
    },
    parser: tsParser,
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      'no-console': ['warn', { allow: ['error', 'warn'] }],
      'prefer-const': 'error',
    },
  },
]
```

✅ Prettier 설정
```json
// .prettierrc
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

✅ VS Code 권장 설정
```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib"
}

// .vscode/extensions.json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "bradlc.vscode-tailwindcss",
    "ms-vscode.vscode-typescript-next"
  ]
}
```

✅ Vitest 설정
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    globals: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      exclude: ['node_modules/', '.next/', 'src/test/'],
      thresholds: { lines: 70, functions: 70 },
    },
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
})
```

## 실전 팁
- `npm run type-check` CI에서 별도 단계로 실행 (빌드와 분리)
- `eslint --cache` 변경된 파일만 검사로 속도 향상
- `.editorconfig` 에디터 무관 기본 설정

## 주의사항
- eslint-disable 주석 남발 금지 — 근본 원인 해결이 우선
- 도구 설정 변경은 팀 합의 후 진행
- Node.js 버전 `.nvmrc`로 고정 — 팀원 간 버전 불일치 방지
