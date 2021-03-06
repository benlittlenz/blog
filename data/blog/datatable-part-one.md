---
title: Build a datatable with Laravel, Vuejs and Tailwind Part 1
date: '2021-03-01'
tags: ['laravel', 'vuejs', 'tailwindcss', 'datatable']
draft: false
summary: In this series, we'll set up and go through the process of building a data-table with Laravel, VueJS and TailwindCSS. We'll be covering a wide range of topics and features such as CRUD operations, searching, advanced filtering, sorting and record limiting. At the end of this series, you will walk away with a good foundation to improve and build upon to meet any further requirements.
---

## Introduction

In this series, we'll set up and go through the process of building a data-table with Laravel, VueJS and TailwindCSS. We'll be covering a wide range of topics and features such as CRUD operations, searching, advanced filtering, sorting and record limiting.

At the end of this series, you will walk away with a good foundation to improve and build upon to meet any further requirements.

Before we start we need to address that creating a data table requires careful planning and consideration. Depending on what kind of application you are building, you need to determine if records should be created or updated inline or in a pop-up modal? Should records be grouped or not grouped? Do you need flexible column sizing?

As you can see, data tables come with a set of challenges. From a design perspective, we need the data table to be easy to use, responsive, readable, have the right colouring, and accessible.

From a functionality perspective, we need to be able to create, read, update and delete data. With all of this data, we need to paginate, search, sort and apply complex filtering options.

Since testing is a huge topic in itself, we will not be covering any testing in this series.

## Tech Stack

This data-table series is written with Laravel and VueJS developers in mind using Tailwindcss for styling and Vuex for our frontend state management. A basic understanding of PHP, Laravel, VueJS and Vuex is essential. An understanding of Tailwind would be beneficial, hwoever not essential.

As a starting point, clone the GitHub repository, as this series does not focus on styling our data-table, and to reduce clutter, we have included the HTML markup and styling for our data table and related components.

## API Endpoints

In our repository starter, we have defined the following API routes to get, create, update and delete resources.

In our `routes/api.php` file we have the following API routes.

| Method |          URI           |                              Description |
| :----- | :--------------------: | ---------------------------------------: |
| GET    |    /api/tranactions    |          List all available transactions |
| POST   |   /api/transactions    |        Create a new transaction resource |
| PATCH  | /api/transactions/{id} | Update a particular transaction resource |
| DELETE | /api/transactions/{id} | Delete a particular transaction resource |

## Setup

To get started, clone the following repository. In here, you will find a few things already set up for you, such as our transaction `migration, model, controller, factory and resource` files. And if you take a look in the `js` folder, you will find a few components markup as well as our `vuex` store and our main data-table component.

`git clone xxxx`

Note: Make sure you have created your database, and updated your `.env` file and migrated the latest changes.

Once you're all setup, let's start off by creating some dummy data using our `Transaction` factory to create 20 records.

`Transaction::factory()->count(20)->create()`

Now that we have a list of transactions we can think about how we are going to fetch this data from our client slide.

Let's head over to our `index` method in our `TransactionController` file where we will return our list of transactions when the method is called.

```php{4}:app/Http/Controllers/TransactionController.php
// ...
public function index()
{
    return TransactionResource::collection(Transaction::latest()->get())->response();
}
// ...
```

## Fetching Our Transactions

Let's start off by creating a new constant in our `mutation-types` file called `FETCH_TRANSACTIONS`

```javascript:resources/js/store/mutation-types
export const FETCH_TRANSACTIONS = 'FETCH_TRANSACTIONS'
```

```javascript{2,3,4}:resources/js/store/modules/transactions
// ...
export const getters = {
  transactions: (state) => state.transactions,
}
// ...
```

```javascript{3,4,5}:resources/js/store/modules/transactions
// ...
export const mutations = {
    [types.FETCH_TRANSACTIONS](state, { transactions }) {
        state.transactions = transactions
    },
},
// ...
```

In the code above, we're creating a `transactions` function which will return out transactions from our state. In our mutation objects, we're creating a mutation function`FETCH_TRANSACTIONS` which takes in our `transactions` and updates the state with our transactions.

```javascript{3,4,5,6,7,8,9,10,11}:resources/js/DataTable.vue
// ...
export const actions = {
    async fetchTransactions({ commit }) {
        try {
            const { data } = await axios.get(`/api/transactions`)

            commit(types.FETCH_TRANSACTIONS, { transactions: data })
        } catch (e) {
            console.log("ERROR", e)
        }
    },
}
// ...
```

In the method above, we're performing an API call to get all the `transactions` from our database, once we get the data back from our API, we commit our `FETCH_TRANSACTIONS` mutation which mutates our state, adding on all of our transactions.

Let's now move onto to our data-table and start referencing these vuex methods we have just setup, and start displaying our `transactions` in our table.


```javascript{6-20}:resources/js/DataTable.vue
export default {
  data: () => ({
    loading: true,
  }),
  mounted() {
    this.fetchTransactions()
  },
  computed: {
    ...mapGetters({
      transactions: 'transactions/transactions',
    }),
  },
  methods: {
    async fetchTransactions() {
      await this.$store.dispatch('transactions/fetchTransactions')
      this.loading = false
    },
  },
}
```

In our `fetchTranactions` method above, we can access the store using `this.$store` which is available in all our of child components, and then reference the `fetchTransactions` action method in our `store` which makes an API call to our database to fetch all transactions, which is called when the component is mounted and sets a loading variable to false to indicate that the data was fetched successfully.

```html{3,4}:resources/js/DataTable.vue
<div>
  <div>
    <div v-if="loading">Loading...</div>
    <div v-else class="bg-white rounded-sm shadow mt-2 overflow-auto max-h-screen">
      <table class="border-collapse w-full whitespace-no-wrap bg-white">
        // ...
      </table>
    </div>
  </div>
</div>
```

We will now need to create a new data property that will contain an array of column headings which we will then loop over and display the column name.

```javascript{4}:resources/js/DataTable.vue
export default {
  data: () => ({
    loading: true,
    columns: ["date", "category", "payee", "amount", "account", "notes"],
  }),
```

```html{5,6}:resources/js/DataTable.vue
<th>
    <input
        class="form-checkbox focus:outline-none focus:shadow-outline"
    />
    </label>
</th>
<th
    v-for="(column, index) in columns"
    :key="index"
    class="w-40 bg-gray-100 sticky top-0 border-solid border border-gray-200 cursor-pointer px-6 py-3 text-gray-600 font-bold tracking-wider uppercase text-xs"
>
  <span>{{ column }}</span>
</th>
```

We are now reading to start outputting our data to our data-table, let's start by looping over our transaction data and display the data in each column.

```html{3,6,10,13,16,19,22,25}:resources/js/DataTable.vue
<tbody>
  <template>
    <tr
      v-for="transaction in transactions.data"
      :key="transaction.id"
      class="max-h-2 hover:bg-gray-50 cursor-pointer"
    >
      <td class="border-dashed border-t border-gray-200 px-3">
        <label
          class="text-teal-500 inline-flex justify-between items-center hover:bg-gray-200 px-2 py-2 rounded-lg cursor-pointer"
        >
          <input
            type="checkbox"
            class="form-checkbox rowCheckbox focus:outline-none focus:shadow-outline"
          />
        </label>
      </td>
      <td class="border-solid border border-gray-200">
        <span class="text-gray-700 px-6 py-1 flex items-center">{{ transaction.date }}</span>
      </td>
      <td class="border-solid border border-gray-200">
        <span class="text-gray-700 px-6 py-1 flex items-center">{{ transaction.category }}</span>
      </td>
      <td class="border-solid border border-gray-200">
        <span class="text-gray-700 px-6 py-1 flex items-center">{{ transaction.payee }}</span>
      </td>
      <td class="border-solid border border-gray-200">
        <span class="text-gray-700 px-6 py-1 flex items-center">${{ transaction.amount }}</span>
      </td>
      <td class="border-solid border border-gray-200">
        <span class="text-gray-700 px-6 py-1 flex items-center">{{ transaction.account }}</span>
      </td>
      <td class="border-solid border border-gray-200">
        <span class="text-gray-700 px-6 py-1 flex items-center">{{ transaction.notes }}</span>
      </td>
      <td class="border-solid border border-gray-200">
        <div class="flex flex-col justify-center items-center">
          <span><icon name="details" /></span>
        </div>
      </td>
    </tr>
  </template>
</tbody>
```

This now concludes part 1 of this series, in which we set up a vuex store to fetch data from our backend and rendering our data inside our table. In the next part, we'll implement a quick search feature, as well as record limiting in our table.