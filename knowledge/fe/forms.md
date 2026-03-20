# Forms — 폼 처리 전략

> 폼은 사용자와 데이터 사이의 계약이다. 검증은 친절하게, 에러는 즉시, 제출은 안전하게.

## 핵심 원칙
- **React Hook Form + Zod**: 성능(비제어) + 타입 안전 검증
- **실시간 검증**: blur 이벤트에서 검증, submit 전체 검증
- **서버 에러 처리**: API 에러를 폼 필드에 연결
- **낙관적 비활성화**: 제출 중 중복 제출 방지

## 코드 예시

✅ 기본 패턴: RHF + Zod
```typescript
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  email: z.string().email('올바른 이메일 형식이 아닙니다'),
  password: z.string().min(8, '비밀번호는 8자 이상이어야 합니다'),
  confirmPassword: z.string(),
}).refine(
  (data) => data.password === data.confirmPassword,
  { message: '비밀번호가 일치하지 않습니다', path: ['confirmPassword'] }
)

type FormValues = z.infer<typeof schema>

function SignupForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting }, setError } = useForm<FormValues>({
    resolver: zodResolver(schema),
  })

  const onSubmit = async (data: FormValues) => {
    try {
      await authApi.signup(data)
    } catch (error) {
      if (error.code === 'EMAIL_TAKEN') {
        setError('email', { message: '이미 사용 중인 이메일입니다' })
      }
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="email">이메일</label>
        <input id="email" {...register('email')} aria-describedby="email-error" />
        {errors.email && <span id="email-error" role="alert">{errors.email.message}</span>}
      </div>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? '가입 중...' : '가입하기'}
      </button>
    </form>
  )
}
```

✅ 멀티스텝 폼
```typescript
function MultiStepForm() {
  const [step, setStep] = useState(1)
  const methods = useForm<FullFormValues>({ resolver: zodResolver(fullSchema) })

  const handleNext = async () => {
    const isValid = await methods.trigger(stepFields[step])
    if (isValid) setStep(s => s + 1)
  }

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        {step === 1 && <PersonalInfoStep />}
        {step === 2 && <AddressStep />}
        {step === 3 && <ReviewStep />}
        <button type="button" onClick={handleNext}>다음</button>
      </form>
    </FormProvider>
  )
}
```

✅ 동적 필드 (useFieldArray)
```typescript
function TagInput() {
  const { control } = useFormContext()
  const { fields, append, remove } = useFieldArray({ control, name: 'tags' })

  return (
    <div>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input {...register(`tags.${index}.value`)} />
          <button type="button" onClick={() => remove(index)}>삭제</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ value: '' })}>태그 추가</button>
    </div>
  )
}
```

## 실전 팁
- `mode: 'onBlur'`로 blur 시 검증 (submit 전 피드백)
- `defaultValues` 반드시 지정 — undefined 방지
- Server Action과 통합 시 `useFormState` 활용

## 주의사항
- Controlled Input으로 대규모 폼 구성 시 성능 저하 — RHF 비제어 방식 사용
- 검증 오류 메시지는 사용자 친화적으로 — "required" 대신 "이름을 입력해주세요"
- 파일 업로드는 별도 처리 — RHF의 `watch`로 미리보기 구현
