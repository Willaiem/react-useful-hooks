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


## usePagination (with react-query)

```ts
const usePagination = <T,>(key: string, asyncFn: (pageIndex: number) => Promise<T>) => {
  const [index, setIndex] = useState(0)
  const { data, isLoading, hasNextPage, hasPreviousPage, fetchNextPage: fetchNext, fetchPreviousPage: fetchPrevious }
    = useInfiniteQuery({
      queryKey: key,
      queryFn: ({ pageParam }) => asyncFn(pageParam || 0),
      getNextPageParam: (lastPage) => lastPage !== undefined || lastPage !== null,
      getPreviousPageParam: () => index > 0,
    })

  const currentData = data?.pages.at(-1)

  const fetchNextPage = () => {
    setIndex(index + 1)
    fetchNext({ pageParam: index + 1 })
  }

  const fetchPreviousPage = () => {
    setIndex(index - 1)
    fetchPrevious({ pageParam: index - 1 })
  }

  return { data: currentData, isLoading, fetchNextPage, fetchPreviousPage, hasNextPage, hasPreviousPage }
}
```
