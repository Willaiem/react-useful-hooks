# react-useful-hooks

Some custom hooks I made.
I want to have some cool hooks in one place to find them easy.

## useAsync (vanilla)

<p>Based on [TanStack Query's useQuery](https://tanstack.com/query/v4/docs/react/guides/queries) hook.</p>
transformToError: https://gist.github.com/Willaiem/4015d7ef8dce550be6863f203c29036f

```ts
import { transformToError } from '../utils/transformToError'

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
    } catch (e) {
      const err = transformToError(e)
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

const fetchTodo = () => fetch('https://jsonplaceholder.typicode.com/todos/1').then(response => response.json())

const Todo = () => {
  const {data: todo, isLoading } = useAsync<TTodo>(fetchTodo)

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
type TTodo = {
  userId: number
  title: string
}

const fetchTodo = () => fetch('https://jsonplaceholder.typicode.com/todos/1').then(response => response.json())

const Todo = () => {
  const state = useAsync<TTodo>(fetchTodo)

  const isLoading = state.type !== 'fulfilled'

  if (isLoading) {
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
    (response) => response.json()
  )

export const Comments = () => {
  const {
    data: comments,
    fetchPreviousPage,
    fetchNextPage,
    hasNextPage,
    hasPreviousPage
  } = usePagination<Comment[]>(fetchComments, {
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

## useVirtual (for [list virtualization](https://www.patterns.dev/posts/virtual-lists))
Based on [TanStack's React Virtual](https://tanstack.com/virtual/v3).

```ts
const getThrottle = (callback: () => void, ms: number) => {
  let isThrottled = false;

  return () => {
    if (isThrottled) return;

    isThrottled = true;
    callback();

    setTimeout(() => {
      isThrottled = false;
    }, ms);
  }
}

const useThrottledForceRender = () => {
  const forceRerender = useState({})[1]

  const RERENDER_DELAY_MS = 20; // you can play with this value to see how it affects the performance
  const throttleRerender = getThrottle(() => {
    forceRerender({})
  }, RERENDER_DELAY_MS)

  return throttleRerender
}

export const useVirtual = ({ count, getScrollElement, estimateSize }: {
  count: number
  getScrollElement: () => HTMLElement | null | undefined
  estimateSize: () => number,
}) => {
  const BUFFOR_SIZE = 2 // for elements that are not visible but are close to the visible area

  const forceRerender = useThrottledForceRender()

  const getTotalSize = () => count * estimateSize()

  const getVirtualItems = () => {
    const scrollElement = getScrollElement();
    if (!scrollElement) return [];

    const scrollTop = scrollElement.scrollTop;
    const scrollBottom = scrollTop + scrollElement.clientHeight;

    const startIndex = Math.floor(scrollTop / estimateSize());
    const endIndex = Math.ceil(scrollBottom / estimateSize());

    const size = estimateSize();

    const elementsToRender = (endIndex - startIndex) + BUFFOR_SIZE;

    return Array(elementsToRender)
      .fill(0)
      .map((_, idx) => ({
        index: startIndex + idx,
        start: size * (startIndex + idx),
        end: size * (startIndex + idx + 1),
        size
      }));
  }

  useEffect(() => {
    const scrollElement = getScrollElement();
    if (!scrollElement) return;

    const handleScroll = () => {
      forceRerender()
    }

    scrollElement.addEventListener('scroll', handleScroll);
    return () => {
      scrollElement.removeEventListener('scroll', handleScroll);
    }
  }, [])

  return {
    getTotalSize,
    getVirtualItems
  }
}
```

Usage (taken from the [React Virtual's docs](https://tanstack.com/virtual/v3/docs/guide/introduction)):
```ts
import { useVirtual } from './useVirtual'
import { useAsync } from './useAsync'

const getPhotos = async () => {
  const response = await fetch("https://jsonplaceholder.typicode.com/photos");
  const photos = await response.json()
  return photos;
};

const PhotosList = () => {
  const {data: photos }= useAsync<Photo[]>(() => getPhotos());

  const containerRef = useRef<ElementRef<'div'>>(null);

  const virtual = useVirtual({
    count: photos?.length ?? 0,
    getScrollElement: () => containerRef.current,
    estimateSize: () => 245
  })

  if (!photos) {
    return <div>Loading...</div>
  }

  return (
    <div>
      <h1>Virtual</h1>
      <div
        ref={containerRef}
        style={{
          height: document.documentElement.clientHeight,
          overflow: 'auto', // Make it scroll!
        }}
      >
        {/* The large inner element to hold all of the items */}
        <div
          style={{
            height: `${virtual.getTotalSize()}px`,
            width: '100%',
            position: 'relative',
          }}
        >
          {/* Only the visible items in the virtualizer, manually positioned to be in view */}
          {virtual.getVirtualItems().map((virtualItem) => (
            <div
              key={virtualItem.index}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <p>{photos[virtualItem.index].title}</p>
              <img src={photos[virtualItem.index].thumbnailUrl} />
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};
```
