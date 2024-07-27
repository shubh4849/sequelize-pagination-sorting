## Introduction

The Paginate Plugin is designed to streamline and enhance the retrieval of list results from databases using Sequelize ORM. It provides robust pagination, filtering, and dynamic inclusion of associated models, ensuring optimal performance and accurate data representation. This document aims to provide an in-depth understanding of how the paginate function works, including detailed explanations of its parameters, SQL queries involved, and examples with dummy tables.

## Table of Contents

1. [Overview](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
2. [Database Tables](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
3. [Controller Logic](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
4. [Paginate Function](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
    - [Parameters](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
    - [Options Construction](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
    - [Finding Results](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
    - [Counting Results](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
5. [SQL Query Explanation](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
6. [Detailed Examples](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)
7. [Best Practices](https://www.notion.so/Paginate-Plugin-Documentation-54d63b1461754bc7af37b12ae6ed6cdb?pvs=21)

## Overview

The paginate function allows you to fetch paginated results from a database table, with support for dynamic filtering and inclusion of associated models. It ensures accurate counting of total results and maintains the integrity of the main query by conditionally applying subqueries.

## Database Tables

### 1. Products Table

- **Fields**:
    - `id`: INT, primary key
    - `name`: VARCHAR(255)
    - `description`: TEXT
    - `companyId`: INT, foreign key
    - `productImageKey`: VARCHAR(255)
    - `productImageUrl`: VARCHAR(255)
    - `isDeleted`: BOOLEAN, default FALSE
    - `createdAt`: DATETIME
    - `updatedAt`: DATETIME

### 2. ProductTypes Table

- **Fields**:
    - `id`: INT, primary key
    - `name`: VARCHAR(255)
    - `icon`: VARCHAR(255)

### 3. ProductProductTypes Table

- **Fields**:
    - `productId`: INT, foreign key references `products(id)`
    - `productTypeId`: INT, foreign key references `product_types(id)`
    - Primary key: (`productId`, `productTypeId`)

### Relationships

- A product can have multiple product types (many-to-many relationship).

## Controller Logic

Here's an example of how the paginate function is used in a controller to search for products:

```

const searchProduct = async (keyword, queryParams, typeIds) => {
  const filters = {
    isDeleted: false,
    ...(keyword && {
      [Op.or]: [{ name: { [Op.like]: `%${keyword}%` } }, { description: { [Op.like]: `%${keyword}%` } }],
    }),
  };
//keyword is optional
  const include = [
    {
      model: ProductProductType,
      as: 'prodTypes',
      include: {
        model: ProductType,
        as: 'productTypeDefined',
        attributes: ['id', 'name', 'icon'],
      },
    },
  ];

  if (typeIds) {
    include[0].include.where = {
      id: { [Op.in]: typeIds },
    };
  }
//typeids are product types that are assigned to products which can be multiple
  const paginationResult = await paginate(Product, queryParams, filters, include, !typeIds);
  return paginationResult;
};

```

## Paginate Function

### Parameters

- **`model`**: The Sequelize model to query.
- **`queryParams`**: An object containing pagination and sorting options (page, limit, sortBy, sortOrder).
- **`filters`**: An object containing filter conditions.
- **`include`**: An array of Sequelize include options for associated models.
- **`subQuery`**: A boolean indicating whether to use a subquery for the main query.

### Options Construction

The `applyListOptions` function constructs pagination and sorting options:

```

const applyListOptions = (queryParams) => {
  const page = parseInt(queryParams.page, 10) || 1;
  const limit = parseInt(queryParams.limit, 10) || 10;
  const sortBy = queryParams.sortBy || 'createdAt';
  const sortOrder = queryParams.sortOrder || 'desc';

  return {
    options: {
      order: [[sortBy, sortOrder]],
      offset: (page - 1) * limit,
      limit,
    },
    page,
    limit,
  };
};

```

### Finding Results

The main query to fetch results from the database:

```

const paginate = async (model, queryParams, filters = {}, include = [], subQuery = true) => {
  try {
    const { options, page, limit } = applyListOptions(queryParams);

    const queryOptions = {
      ...options,
      ...(Object.keys(filters).length > 0 && { where: filters }),
      ...(include.length > 0 && { include }),
      distinct: true,
      subQuery,
    };

    const rows = await model.findAll(queryOptions);

```

### Counting Results

A separate query to count the total number of results:

```

    const countOptions = {
      where: filters,
      distinct: true,
      col: 'id',
      ...(include.length > 0 && {
        include: include.map((inc) => ({
          ...inc,
          duplicating: false,
          attributes: [],
        })),
      }),
    };

    const count = await model.count(countOptions);
    const totalPages = Math.ceil(count / limit);

    return {
      results: rows,
      page,
      limit,
      totalPages,
      totalResults: count,
    };
  } catch (error) {
    throw new Error(error.message);
  }
};

```

## SQL Query Explanation

### Applying List Options

The `applyListOptions` function generates pagination and sorting options:

- **Pagination**: Calculating the `offset` and `limit` for the SQL query.
- **Sorting**: Specifying the `order` clause for the SQL query.

**Example:**

```sql

ORDER BY createdAt DESC
LIMIT 10 OFFSET 0

```

### Building Query Options

The main query options are constructed based on the provided filters and includes:

**Example:**

```

const queryOptions = {
  order: [['createdAt', 'DESC']],
  offset: 0,
  limit: 10,
  where: { isDeleted: false, [Op.or]: [{ name: { [Op.like]: `%keyword%` } }, { description: { [Op.like]: `%keyword%` } }] },
  include: [
    {
      model: ProductProductType,
      as: 'prodTypes',
      include: {
        model: ProductType,
        as: 'productTypeDefined',
        attributes: ['id', 'name', 'icon'],
        where: { id: { [Op.in]: [2, 4, 8, 9] } },
      },
    },
  ],
  distinct: true,
  subQuery: false,
};

```

### SQL Query Construction

**Finding Results:**

```sql

SELECT DISTINCT
  products.*,
  product_product_types.*,
  product_types.*
FROM
  products
LEFT JOIN product_product_types ON products.id = product_product_types.productId
LEFT JOIN product_types ON product_product_types.productTypeId = product_types.id
WHERE
  products.isDeleted = FALSE
  AND (products.name LIKE '%keyword%' OR products.description LIKE '%keyword%')
ORDER BY
  products.createdAt DESC
LIMIT 10 OFFSET 0;

```

**Counting Results:**

```sql

SELECT COUNT(DISTINCT products.id) AS count
FROM
  products
LEFT JOIN product_product_types ON products.id = product_product_types.productId
LEFT JOIN product_types ON product_product_types.productTypeId = product_types.id
WHERE
  products.isDeleted = FALSE
  AND (products.name LIKE '%keyword%' OR products.description LIKE '%keyword%');

```

## Best Practices

1. **Dynamic Filtering**: Ensure filters and includes are applied only when necessary to optimize query performance.
2. **Conditional Subqueries**: Use `subQuery: false` only when filtering by associations to maintain result accuracy.
3. **Separation of Concerns**: Separate the logic for finding results and counting total results to improve clarity and maintainability.

By following this guide, you can effectively implement the paginate function to handle complex queries with dynamic filtering and associations, ensuring accurate and efficient data retrieval in your projects.
