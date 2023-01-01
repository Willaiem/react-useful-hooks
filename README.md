# react-useful-hooks
Some custom hooks I made.

## useAsync

```ts
const useAsync = <T,>(asyncFn: () => Promise<T>) => {
  const [status, setStatus] = useState<'idle' | 'pending' | 'success' | 'error'>('idle')
  const [data, setData] = useState<T | undefined>(undefined)
  const [error, setError] = useState<Error | null>(null)

  const isLoading = status === 'pending'

  const execute = useCallback(async () => {
    setStatus('pending')
    setData(undefined)
    setError(null)

    try {
      const response = await asyncFn()
      setData(response)
      setStatus('success')
    } catch (err) {
      setError(err)
      setStatus('error')
    }

  }, [asyncFn])

  useEffect(() => {
    execute()
  }, [execute])

  return { status, data, error, isLoading }
}
```
