To use Redux with React for CRUD operations on a `Product` model in a TypeScript project, follow these steps. The `Product` model includes the following fields: `SKU`, `name`, `price`, `quantity`, and `total`. This guide will walk you through setting up Redux, defining actions, reducers, and integrating them into a React component.

### 1. Install Required Packages

Make sure you have the following packages installed:

```bash
npm install redux react-redux @reduxjs/toolkit axios
npm install --save-dev @types/react-redux
```

### 2. Define the `Product` Model

Create a `types.ts` file for your `Product` type:

```ts
// src/types.ts
export interface Product {
  SKU: string;
  name: string;
  price: number;
  quantity: number;
  total: number;
}
```

### 3. Create the Redux Slice

We'll use Redux Toolkit to create a slice for product-related state and actions:

```ts
// src/features/products/productSlice.ts
import { createSlice, PayloadAction, createAsyncThunk } from '@reduxjs/toolkit';
import { Product } from '../../types';
import axios from 'axios';

interface ProductState {
  products: Product[];
  loading: boolean;
  error: string | null;
}

const initialState: ProductState = {
  products: [],
  loading: false,
  error: null,
};

// Async actions for CRUD operations
export const fetchProducts = createAsyncThunk('products/fetch', async () => {
  const response = await axios.get<Product[]>('/api/products');
  return response.data;
});

export const addProduct = createAsyncThunk('products/add', async (product: Product) => {
  const response = await axios.post<Product>('/api/products', product);
  return response.data;
});

export const updateProduct = createAsyncThunk('products/update', async (product: Product) => {
  const response = await axios.put<Product>(`/api/products/${product.SKU}`, product);
  return response.data;
});

export const deleteProduct = createAsyncThunk('products/delete', async (SKU: string) => {
  await axios.delete(`/api/products/${SKU}`);
  return SKU;
});

const productSlice = createSlice({
  name: 'products',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      // Fetch products
      .addCase(fetchProducts.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchProducts.fulfilled, (state, action: PayloadAction<Product[]>) => {
        state.loading = false;
        state.products = action.payload;
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to fetch products';
      })
      // Add product
      .addCase(addProduct.pending, (state) => {
        state.loading = true;
      })
      .addCase(addProduct.fulfilled, (state, action: PayloadAction<Product>) => {
        state.loading = false;
        state.products.push(action.payload);
      })
      .addCase(addProduct.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to add product';
      })
      // Update product
      .addCase(updateProduct.pending, (state) => {
        state.loading = true;
      })
      .addCase(updateProduct.fulfilled, (state, action: PayloadAction<Product>) => {
        state.loading = false;
        const index = state.products.findIndex((p) => p.SKU === action.payload.SKU);
        if (index !== -1) {
          state.products[index] = action.payload;
        }
      })
      .addCase(updateProduct.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to update product';
      })
      // Delete product
      .addCase(deleteProduct.pending, (state) => {
        state.loading = true;
      })
      .addCase(deleteProduct.fulfilled, (state, action: PayloadAction<string>) => {
        state.loading = false;
        state.products = state.products.filter((product) => product.SKU !== action.payload);
      })
      .addCase(deleteProduct.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to delete product';
      });
  },
});

export default productSlice.reducer;
```

### 4. Set Up Redux Store

Configure the Redux store in `store.ts`:

```ts
// src/app/store.ts
import { configureStore } from '@reduxjs/toolkit';
import productReducer from '../features/products/productSlice';

export const store = configureStore({
  reducer: {
    products: productReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### 5. Provide the Store to React

Wrap your application with the Redux `Provider` in `index.tsx`:

```tsx
// src/index.tsx
import React from 'react';
import ReactDOM from 'react-dom';
import { Provider } from 'react-redux';
import { store } from './app/store';
import App from './App';

ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
);
```

### 6. Create a React Component to Use Redux for CRUD

Now, let's create a component that fetches, adds, updates, and deletes products:

```tsx
// src/components/ProductList.tsx
import React, { useEffect, useState } from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { RootState, AppDispatch } from '../app/store';
import { fetchProducts, addProduct, updateProduct, deleteProduct } from '../features/products/productSlice';
import { Product } from '../types';

const ProductList: React.FC = () => {
  const dispatch = useDispatch<AppDispatch>();
  const { products, loading, error } = useSelector((state: RootState) => state.products);
  const [newProduct, setNewProduct] = useState<Product>({
    SKU: '',
    name: '',
    price: 0,
    quantity: 0,
    total: 0,
  });

  useEffect(() => {
    dispatch(fetchProducts());
  }, [dispatch]);

  const handleAddProduct = () => {
    dispatch(addProduct(newProduct));
  };

  const handleUpdateProduct = (product: Product) => {
    dispatch(updateProduct(product));
  };

  const handleDeleteProduct = (SKU: string) => {
    dispatch(deleteProduct(SKU));
  };

  return (
    <div>
      <h1>Product List</h1>
      {loading && <p>Loading...</p>}
      {error && <p>Error: {error}</p>}
      <ul>
        {products.map((product) => (
          <li key={product.SKU}>
            {product.name} - {product.price} - {product.quantity} - {product.total}
            <button onClick={() => handleUpdateProduct({ ...product, quantity: product.quantity + 1 })}>Update</button>
            <button onClick={() => handleDeleteProduct(product.SKU)}>Delete</button>
          </li>
        ))}
      </ul>
      <h2>Add Product</h2>
      <input
        type="text"
        value={newProduct.name}
        placeholder="Name"
        onChange={(e) => setNewProduct({ ...newProduct, name: e.target.value })}
      />
      <input
        type="number"
        value={newProduct.price}
        placeholder="Price"
        onChange={(e) => setNewProduct({ ...newProduct, price: Number(e.target.value) })}
      />
      <input
        type="number"
        value={newProduct.quantity}
        placeholder="Quantity"
        onChange={(e) => setNewProduct({ ...newProduct, quantity: Number(e.target.value) })}
      />
      <button onClick={handleAddProduct}>Add Product</button>
    </div>
  );
};

export default ProductList;
```

### 7. Adjust API Endpoints

Make sure your backend API is properly set up to handle the `/api/products` CRUD operations.

### 8. Run the Application

Start your application and verify that you can perform CRUD operations on the `Product` model.
