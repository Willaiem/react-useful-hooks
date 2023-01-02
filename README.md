# react-useful-hooks
Some custom hooks I made.
I want to have some cool hooks in one place to find them easy.

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

## usePagination (vanilla)
This is basically useAsync, but with index and previous/next page support.
```ts
const usePagination = <T,>(asyncFn: (pageIndex: number) => Promise<T>) => {
  const [index, setIndex] = useState(0)
  const [status, setStatus] = useState<'idle' | 'pending' | 'success' | 'error'>('idle')
  const [data, setData] = useState<T | undefined>(undefined)
  const [error, setError] = useState<Error | null>(null)

  const hasPreviousPage = index > 0
  const hasNextPage = index > 0 && (data !== undefined || data !== null) 
  const isLoading = status === 'pending'

  const execute = useCallback(async (idx: number) => {
    setStatus('pending')
    setData(undefined)
    setError(null)

    try {
      const response = await asyncFn(idx)
      setData(response)
      setStatus('success')
    } catch (err) {
      setError(err)
      setStatus('error')
    }
    
  }, [asyncFn, index])

  useEffect(() => {
    execute(index)
  }, [execute, index])

  const fetchNextPage = () => {
    setIndex(index + 1)
  }

  const fetchPreviousPage = () => {
    setIndex(index - 1)
  }

  return { data, error, status, fetchNextPage, fetchPreviousPage, hasPreviousPage, hasNextPage, isLoading }
}
```

## usePagination (with react-query)

```ts
const usePagination = <T,>(key: string, asyncFn: (pageIndex: number) => Promise<T>) => {
  const [index, setIndex] = useState(0)
  const { data, isFetching, isFetchingNextPage, isFetchingPreviousPage, hasNextPage, hasPreviousPage, fetchNextPage: fetchNext, fetchPreviousPage: fetchPrevious }
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

  const isLoading = isFetching && (isFetchingNextPage || isFetchingPreviousPage)

  return { data: currentData, isLoading, fetchNextPage, fetchPreviousPage, hasNextPage, hasPreviousPage }
}
```

Short explanation:
- I assume that ``lastPage`` will be either ``undefined`` or ``null`` when we will be at the last page.
