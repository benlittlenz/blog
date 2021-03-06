---
title: Supabase Data Table
date: '2021-05-29'
tags: ['supabase', 'data-table', 'react', 'superbase tutorial']
draft: false
summary: Let's review creating one-to-one, one-to-many and many-to-many relationships with Supabase.
---

# Introduction

In this tutorial, we'll set up a simple data-table with pagination using ReactJS, Supabase and TailwindCSS for styling.

This tutorial assumes you have a basic understanding of how supabase works and a project already setup. If not click here https://supabase.io/docs/guides/with-react to view the offical quick-start guide using React.

Here is what we will be building:

![table-demo](https://user-images.githubusercontent.com/39175284/120092175-26199300-c165-11eb-8d6a-f5c1c9bf93be.gif)

In this example, we will be rendering a list of posts and displaying the posts in a table with pagination. This post table looks like so:

```markup
  posts
    id - int8
    added - date
    content - text
    created_by - int8

  users
    id - int8
    name - text
```


Let's start off by creating a `Table` component and create a a function to fetch posts from our database.

```jsx:src/components/Table
export default function Table() {
  const [posts, setPosts] = useState([]);
  useEffect(() => {
    fetchPosts();
  }, []);

  const fetchPosts = async () => {
    let { data, error } = await supabase
      .from("posts")
      .select(`
            id, added, content,
            users (name)
      `)
      .order('id', { ascending: true })
    console.log(data)
    setPosts(data)
    if (error) console.log("error", error);
  };
}
```

In the code above, when the component mounts in the `useEffect` function, we are calling a function `fetchPosts` which queries our `posts` table selecting the `id`, `added`, `content` and the users `name` for the post, setting this data to our local state.

Next up, let's create our table markup and render each post to our data.

```jsx:src/components/Table
  return (
    <div className="flex flex-col h-screen">
      <div className="-my-2 h-2/3 overflow-auto sm:-mx-6 lg:-mx-8">
        <div className="py-2 align-middle inline-block min-w-full sm:px-6 lg:px-8">
          <div className="shadow overflow-hidden border-b border-gray-200 sm:rounded-lg">
            <table className="min-w-full divide-y divide-gray-200">
              <thead className="bg-gray-50">
                <tr>
                <th
                    scope="col"
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
                  >
                    ID
                </th>
                  <th
                    scope="col"
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
                  >
                    Created
                </th>
                  <th
                    scope="col"
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
                  >
                    Content
                </th>
                  <th
                    scope="col"
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
                  >
                    Created By
                </th>
                  <th scope="col" className="relative px-6 py-3">
                    <span className="sr-only">Edit</span>
                  </th>
                </tr>
              </thead>
              <tbody className="bg-white divide-y divide-gray-200">
                {posts.map(post => (
                  <tr key={post.id}>
                    <td className="px-6 py-4 whitespace-nowrap">
                      <div className="flex items-center">
                        <div className="ml-4">
                          <div className="text-sm text-gray-900">{post.id}</div>
                        </div>
                      </div>
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap">
                      <div className="flex items-center">
                        <div className="ml-4">
                          <div className="text-sm text-gray-900">{post.added}</div>
                        </div>
                      </div>
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap">
                      <div className="text-sm text-gray-900">{post.content}</div>
                    </td>

                    <td className="px-6 py-4 whitespace-nowrap">
                      <div className="text-sm text-gray-900">{post.users.name}</div>
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                      <a href="#" className="text-indigo-600 hover:text-indigo-900">
                        Edit
                    </a>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  )
```

Depending on how much data you currently have in your database, this render may be a slow process. We should however display this data in "chunks" so the user can navigate through this data, instead of displaying all the posts at once.

To accomplish this, we should fetch the total count of posts, and then only fetch a small amount of posts, based on what page of the table the user is currently on.

```jsx{6,9-14}:src/components/Table
export default function Table() {
  const [posts, setPosts] = useState([]);
  const [count, setCount] = useState(0);

  useEffect(() => {
    fetchCount().then(() => fetchPosts())
  }, []);

  const fetchCount = async () => {
    const { count } = await supabase
      .from('posts')
      .select('*', { count: 'exact' })
    setCount(count)
  }
```

In the example above, we have created a new function `fetchCount` which selects all records from our `posts` table returning the exact count and setting this value to local state. This function is then being called in the `useEffect` hook first, then fetching posts after.

We next need to track two pieces of data, the current page the user is on, and the number of records to show for each page.

```jsx{4,5}:src/components/Table
export default function Table() {
  const [posts, setPosts] = useState([]);
  const [count, setCount] = useState(0);
  const [currentPage, setCurrentPage] = useState(1);
  const [perPage, setPerPage] = useState(25);

  // ...
}
```
And modify our useEffect function to refetch our posts when a page has been changed.

```jsx{3}:src/components/Table
  useEffect(() => {
    fetchCount().then(() => fetchPosts())
  }, [currentPage]);
```

Back in our `fetchPosts` function, we need to specify the range of records to fetch, we can do this accomplish this by using the `range` function and specifying a start and end index.

```jsx:src/components/Table
  const fetchPosts = async () => {
    const startIndex = (currentPage * perPage) - perPage
    const endIndex = (currentPage * perPage) - 1
    let { data, error } = await supabase
      .from("posts")
      .select(`
            id, added, content,
            users (name)
      `)
      .range(startIndex, endIndex)
      .order('id', { ascending: true })
    setPosts(data)
    if (error) console.log("error", error);
  };
```

The start index is calculated by multiplying the current page by number of records per page subtracting the number of records per page (i.e. (1 * 25) - 25 = 0).

The end index is calculated by multiplying the current page by records per page minus 1. (i.e. (1 * 25) - 1) = 24.

This will fetch 25 records per page.

Next, we're ready to create out pagination component with the following markup:

```jsx:src/components/Pagination
export default function Pagination() {
  return (
    <div className="flex justify-between">

      <div className="flex items-center">
        <div className="">
          <a
            className="mr-4 px-2 py-2 inline-flex items-center cursor-pointer text-sm leading-5 font-medium hover:text-gray-800 hover:bg-gray-100 rounded-lg text-gray-500 focus:outline-none transition ease-in-out duration-150"
          >
            Previous
      </a>
        </div>
        <ul className="flex pl-0 list-none rounded my-2">
          <li className="cursor-pointer">
            <a
              className="px-3 py-2 text-center inline-flex items-center text-sm leading-5 font-medium hover:text-gray-800 hover:bg-gray-100 rounded-lg focus:outline-none transition ease-in-out duration-150"
            >
            </a>
          </li>
        </ul>
        <div className="flex justify-end">
          <a
            className="px-3 py-2 text-center inline-flex items-center text-sm leading-5 font-medium hover:text-gray-800 hover:bg-gray-100 rounded-lg focus:outline-none transition ease-in-out duration-150"
          >
            Next
          </a>
        </div>
      </div>
    </div>
  )
}
```

Passing through the page count, posts per page, current page and a function to set the current page.

```jsx:src/components/Pagination
    /* ... */
    </div>
      <Pagination
        count={count}
        perPage={perPage}
        currentPage={currentPage}
        setCurrentPage={setCurrentPage}
      />
    </div>
  )
}
```
We can calculate the page count by dividing the posts count by records per page and using `Math.ceil` to round the number to the nearest integer.

```jsx{1,2}:src/components/Pagination
export default function Pagination({ count, perPage, currentPage, setCurrentPage }) {
  const pageCount = Math.ceil(count / perPage);

  // ...
}
```

Now that we have our page count, we can create an array of numbers and render a list of pages depending on the record count increasing by +1 as 0 is the starting index

```jsx{2,3,5,8}:src/components/Pagination
  <ul className="flex pl-0 list-none rounded my-2">
    {[...Array(pageCount)].map((e, page) => (
      <li className="cursor-pointer" key={page + 1}>
        <a
          className="px-3 py-2 text-center inline-flex items-center text-sm leading-5 font-medium hover:text-gray-800 hover:bg-gray-100 rounded-lg focus:outline-none transition ease-in-out duration-150"
        >
          {page + 1}
        </a>
      </li>
    ))}
  </ul>
```

Next, we create a function `changePage` which takes in a page number and sets the current page back in our table component which will fetch new posts from out database.

```jsx{4-9}:src/components/Pagination
export default function Pagination({ count, perPage, currentPage, setCurrentPage }) {
  const pageCount = Math.ceil(count / perPage);

  const changePage = (page) => {
    if (page <= 0 || page > pageCount) {
      return;
    }
    setCurrentPage(page);
  }
}
```

Now we can call this `changePage` method and adjust the current page by incrementing / decrementing.

```jsx{5,13,14,21}:src/components/Pagination
  return (
    <div className="flex justify-between">
      <div className="flex items-center">
        <div className="">
          <a
            onClick={() => changePage(currentPage - 1)}
            className={`mr-4 px-2 py-2 inline-flex items-center ${currentPage === 1 ? ' cursor-not-allowed' : ' cursor-pointer '} text-sm leading-5 font-medium hover:text-gray-800 hover:bg-gray-100 rounded-lg text-gray-500 focus:outline-none transition ease-in-out duration-150`}
          >
            Previous
          </a>
        </div>
        <ul className="flex pl-0 list-none rounded my-2">
          {[...Array(pageCount)].map((e, page) => (
            <li className="cursor-pointer" key={page + 1}>
              <a
                onClick={() => changePage(page + 1)}
                className="px-3 py-2 text-center inline-flex items-center text-sm leading-5 font-medium hover:text-gray-800 hover:bg-gray-100 rounded-lg focus:outline-none transition ease-in-out duration-150"
              >
                {page + 1}
              </a>
            </li>

          ))}
        </ul>
        <div className="flex justify-end">
          <a
            onClick={() => changePage(currentPage + 1)}
            className={`ml-4 px-2 py-2 inline-flex items-center ${currentPage === pageCount ? ' cursor-not-allowed' : ' cursor-pointer '} text-sm leading-5 font-medium hover:text-gray-800 hover:bg-gray-100 rounded-lg text-gray-500 focus:outline-none transition ease-in-out duration-150`}
          >
            Next
          </a>
        </div>
      </div>
    </div>

  )
```

Finally, we can add a start and end index to show what records we are displaying in the table (i.e. 1 to 25).
```jsx{3,4}:src/components/Pagination
export default function Pagination({ count, perPage, currentPage, setCurrentPage }) {
  const pageCount = Math.ceil(count / perPage);
  const startIndex = ((currentPage * perPage) - perPage) + 1;
  const endIndex = (currentPage * perPage);

  // ...
```

```jsx{3}:src/components/Pagination
  return (
    <div className="flex justify-between">
      <div className="my-4"><p>Showing {startIndex} to {endIndex}</p></div>
      {/* ... */}
```