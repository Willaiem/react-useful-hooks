# react-useful-hooks

Some custom hooks I made.
I want to have some cool hooks in one place to find them easy.

## useAsync (vanilla)

<p>Based on TanStack Query's useQuery hook.

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

Usage:

```ts
type TTodo = {
  userId: number
  title: string
}

const fetchTodo = () => fetch('https://jsonplaceholder.typicode.com/todos/1').then(response => response.json() as Promise<TTodo>)

const Todo = () => {
  const {data: todo, isLoading } = useAsync(fetchTodo)

  if (isLoading || !todo) {
    return <p>Loading...</p>
  }

  return (
    <div>
        <h1>{todo.title}</h1>
        <p>User id: {todo.userId}</p>
    </div>
  )
}
```

### Alternative - useAsync (with Union State Machine pattern)

This makes the hook easier to read and more typesafe.

```ts
type UseAsync<T> = 
  | {
    type: 'init';
  }
  | {
    type: 'pending'
  }
  | {
    type: 'fulfilled',
    data: T
  }
  | {
    type: 'rejected'
    error: unknown
  }

const useAsync = <T,>(asyncFn: () => Promise<T>) => {
  const [state, setAsyncState] = useState<UseAsync<T>>({
    type: 'init'
  });

  const execute = useCallback(async () => {
    setAsyncState({ type: 'pending' });

    try {
      const response = await asyncFn();
      setAsyncState({ type: 'fulfilled', data: response });
    } catch (err) {
      setAsyncState({ type: 'rejected', error: err });
    }
  }, [asyncFn]);

  useEffect(() => {
    execute();
  }, [execute]);

  return state
};
```

Usage:

```ts
type TUser = {
  name: string
  age: number
}

const fetchUser = () => fetch('https://user.example').then(response => response.json() as Promise<TUser>)

const User = () => {
  const state = useAsync(fetchUser)

  const isLoading = state.type !== 'fulfilled'

  if (isLoading) {
    return <p>Loading...</p>
  }

  return (
    <div>
      <h1>{state.data.name}</h1>
      <p>Age: {state.data.age}</p>
    </div>
  )
}
```

## usePagination (vanilla)

This is basically useAsync, but with index and previous/next page support.
It also support the persisting the data between pages, just like in TanStack Query.

```ts
const usePagination = <T,>(
  asyncFn: (pageIndex: number) => Promise<T>,
  options?: { persistDataBetweenPages?: boolean; initialIndex?: number }
) => {
  const initialIndex = options?.initialIndex ?? 0
  
  const [index, setIndex] = useState(initialIndex)
  const [status, setStatus] = useState<
    "idle" | "pending" | "success" | "error"
  >("idle")
  const [data, setData] = useState<T | undefined>(undefined)
  const [error, setError] = useState<Error | null>(null)

  const hasPreviousPage = index > initialIndex
  const hasNextPage = index >= initialIndex && (data !== undefined || data !== null)
  const isLoading = status === "pending"

  const execute = useCallback(
    async (idx: number) => {
      setStatus("pending")
      setError(null)
      if (!options?.persistDataBetweenPages) {
        setData(undefined)
      }
      
      try {
        const response = await asyncFn(idx)
        setData(response)
        setStatus("success")
      } catch (err: any) {
        setError(err);
        setStatus("error")
      }
    },
    [asyncFn, index]
  )

  useEffect(() => {
    execute(index)
  }, [execute, index])

  const fetchNextPage = () => {
    setIndex(index + 1)
  }

  const fetchPreviousPage = () => {
    setIndex(index - 1)
  }

  return {
    data,
    error,
    status,
    fetchNextPage,
    fetchPreviousPage,
    hasPreviousPage,
    hasNextPage,
    isLoading,
  }
}
```

Usage:

```ts
type Comment = {
  postId: number
  id: number
  name: string
}

const fetchComments = (index: number) =>
  fetch(`https://jsonplaceholder.typicode.com/comments?postId=${index}`).then(
    (response) => response.json() as Promise<Comment[]>
  )

export const Comments = () => {
  const {
    data: comments,
    fetchPreviousPage,
    fetchNextPage,
    hasNextPage,
    hasPreviousPage
  } = usePagination(fetchComments, {
    initialIndex: 1,
    persistDataBetweenPages: true
  })

  return (
    <div>
      {!comments && <p>Loading...</p>}
      {comments && comments.map((comment) => (
        <div key={comment.id}>
          <h2>{comment.name}</h2>
          <p>Post id: {comment.postId}</p>
        </div>
      ))}

      <div>
        <button onClick={fetchPreviousPage} disabled={!hasPreviousPage}>Previous</button>
        <button onClick={fetchNextPage} disabled={!hasNextPage}>Next</button>
      </div>
    </div>
  );
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

- I assume that ``lastPage`` will be either ``undefined`` or ``null`` when we will be at the last page. You can customize it to your needs.
