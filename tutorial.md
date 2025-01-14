# React Hooks Tutorial: Building a Book Library Manager

## Project Overview
We'll build a Book Library Manager that allows users to:
- View a list of books
- Add new books
- Search for books
- Mark books as favorites
- Track reading progress

This project will demonstrate the practical use of useState, useEffect, useRef, and axios.

## Prerequisites
- Basic JavaScript knowledge
- Node.js installed
- npm or yarn package manager
- Basic understanding of HTML/CSS

## Section 1: Project Setup

First, create a new React project:

```bash
npx create-react-app book-library-manager
cd book-library-manager
npm install axios
```

## Section 2: Understanding useState

### What is useState?
useState is a React Hook that allows functional components to manage state. It returns an array with two elements:
1. The current state value
2. A function to update that value

Let's implement our first component using useState:

```jsx
// src/components/BookList.js
import React, { useState } from 'react';

function BookList() {
  // Initialize state with an empty array
  const [books, setBooks] = useState([]);
  const [newBook, setNewBook] = useState({ title: '', author: '' });

  const handleAddBook = () => {
    if (newBook.title && newBook.author) {
      setBooks([...books, { ...newBook, id: Date.now() }]);
      setNewBook({ title: '', author: '' }); // Reset form
    }
  };

  return (
    <div>
      <div>
        <input
          type="text"
          placeholder="Book title"
          value={newBook.title}
          onChange={(e) => setNewBook({ ...newBook, title: e.target.value })}
        />
        <input
          type="text"
          placeholder="Author"
          value={newBook.author}
          onChange={(e) => setNewBook({ ...newBook, author: e.target.value })}
        />
        <button onClick={handleAddBook}>Add Book</button>
      </div>

      <ul>
        {books.map(book => (
          <li key={book.id}>
            {book.title} by {book.author}
          </li>
        ))}
      </ul>
    </div>
  );
}

export default BookList;
```

### Key Points About useState:
1. State updates are asynchronous
2. Previous state can be accessed using the functional update form
3. State should be treated as immutable

## Section 3: Understanding useEffect

### What is useEffect?
useEffect is a Hook that manages side effects in functional components. It runs after every render and can be configured to run only when specific dependencies change.

Let's add data persistence to our BookList component:

```jsx
// src/components/BookList.js
import React, { useState, useEffect } from 'react';

function BookList() {
  const [books, setBooks] = useState([]);
  const [newBook, setNewBook] = useState({ title: '', author: '' });

  // Load books from localStorage on component mount
  useEffect(() => {
    const savedBooks = localStorage.getItem('books');
    if (savedBooks) {
      setBooks(JSON.parse(savedBooks));
    }
  }, []); // Empty dependency array means this runs once on mount

  // Save books to localStorage whenever books state changes
  useEffect(() => {
    localStorage.setItem('books', JSON.stringify(books));
  }, [books]); // This effect runs whenever books state changes

  // ... rest of the component
}
```

### useEffect Cleanup
Sometimes we need to clean up effects (like event listeners or subscriptions):

```jsx
// src/components/ReadingTimer.js
import React, { useState, useEffect } from 'react';

function ReadingTimer() {
  const [readingTime, setReadingTime] = useState(0);
  const [isReading, setIsReading] = useState(false);

  useEffect(() => {
    let timer;
    if (isReading) {
      timer = setInterval(() => {
        setReadingTime(prev => prev + 1);
      }, 1000);
    }
    
    // Cleanup function
    return () => {
      if (timer) {
        clearInterval(timer);
      }
    };
  }, [isReading]);

  return (
    <div>
      <p>Reading Time: {readingTime} seconds</p>
      <button onClick={() => setIsReading(!isReading)}>
        {isReading ? 'Pause' : 'Start'} Reading
      </button>
    </div>
  );
}
```

## Section 4: Working with Axios

### What is Axios?
Axios is a promise-based HTTP client that makes it easy to send HTTP requests and handle responses.

Let's create a service to fetch books from an API:

```jsx
// src/services/bookApi.js
import axios from 'axios';

const API_KEY = 'your_google_books_api_key';
const BASE_URL = 'https://www.googleapis.com/books/v1/volumes';

export const searchBooks = async (query) => {
  try {
    const response = await axios.get(`${BASE_URL}?q=${query}&key=${API_KEY}`);
    return response.data.items || [];
  } catch (error) {
    console.error('Error searching books:', error);
    return [];
  }
};

// src/components/BookSearch.js
import React, { useState, useEffect } from 'react';
import { searchBooks } from '../services/bookApi';

function BookSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    const fetchBooks = async () => {
      if (query.length < 3) return;
      
      setLoading(true);
      try {
        const books = await searchBooks(query);
        setResults(books);
      } catch (error) {
        console.error('Error:', error);
      } finally {
        setLoading(false);
      }
    };

    const timeoutId = setTimeout(fetchBooks, 500); // Debounce search
    return () => clearTimeout(timeoutId);
  }, [query]);

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search books..."
      />
      
      {loading && <p>Loading...</p>}
      
      <ul>
        {results.map(book => (
          <li key={book.id}>
            {book.volumeInfo.title}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## Section 5: Understanding useRef

### What is useRef?
useRef is a Hook that provides a mutable ref object whose .current property is initialized with the passed argument. It persists between renders and doesn't cause re-renders when changed.

Let's implement a reading progress tracker:

```jsx
// src/components/ReadingProgress.js
import React, { useState, useRef, useEffect } from 'react';

function ReadingProgress({ bookContent }) {
  const [progress, setProgress] = useState(0);
  const contentRef = useRef(null);
  const observerRef = useRef(null);

  useEffect(() => {
    // Set up Intersection Observer
    observerRef.current = new IntersectionObserver(
      (entries) => {
        const [entry] = entries;
        const visiblePercentage = 
          (entry.intersectionRect.height / entry.boundingClientRect.height) * 100;
        setProgress(Math.round(visiblePercentage));
      },
      { threshold: Array.from({ length: 101 }, (_, i) => i / 100) }
    );

    if (contentRef.current) {
      observerRef.current.observe(contentRef.current);
    }

    // Cleanup
    return () => {
      if (observerRef.current) {
        observerRef.current.disconnect();
      }
    };
  }, []);

  return (
    <div>
      <div className="progress-bar">
        <div 
          className="progress-fill"
          style={{ width: `${progress}%` }}
        />
      </div>
      
      <div ref={contentRef} className="book-content">
        {bookContent}
      </div>
    </div>
  );
}
```

### Common useRef Use Cases:
1. Accessing DOM elements directly
2. Storing mutable values that don't require re-renders
3. Keeping track of previous values
4. Managing timers and intervals

## Putting It All Together

Now let's create a main App component that combines all these features:

```jsx
// src/App.js
import React, { useState } from 'react';
import BookList from './components/BookList';
import BookSearch from './components/BookSearch';
import ReadingProgress from './components/ReadingProgress';
import ReadingTimer from './components/ReadingTimer';

function App() {
  const [selectedBook, setSelectedBook] = useState(null);

  return (
    <div className="app">
      <h1>Book Library Manager</h1>
      
      <div className="search-section">
        <h2>Search Books</h2>
        <BookSearch onBookSelect={setSelectedBook} />
      </div>
      
      <div className="library-section">
        <h2>My Library</h2>
        <BookList />
      </div>
      
      {selectedBook && (
        <div className="reading-section">
          <h2>Reading: {selectedBook.volumeInfo.title}</h2>
          <ReadingTimer />
          <ReadingProgress 
            bookContent={selectedBook.volumeInfo.description} 
          />
        </div>
      )}
    </div>
  );
}

export default App;
```

## Best Practices and Tips

1. **useState Best Practices:**
   - Keep state minimal and organized
   - Use multiple useState calls for unrelated state
   - Consider using useReducer for complex state logic

2. **useEffect Best Practices:**
   - Always include dependencies array
   - Keep effects focused and single-purpose
   - Clean up subscriptions and listeners
   - Avoid infinite loops

3. **Axios Best Practices:**
   - Create API service modules
   - Handle errors properly
   - Use request/response interceptors
   - Configure defaults

4. **useRef Best Practices:**
   - Don't overuse refs
   - Clean up refs in useEffect cleanup
   - Use TypeScript for better type safety
   - Avoid storing values that should trigger re-renders

## Common Pitfalls to Avoid

1. **State Updates:**
   ```jsx
   // Wrong
   setCount(count + 1);
   setCount(count + 1);

   // Correct
   setCount(prev => prev + 1);
   setCount(prev => prev + 1);
   ```

2. **Effect Dependencies:**
   ```jsx
   // Wrong
   useEffect(() => {
     document.title = `Count: ${count}`;
   }); // Missing dependency array

   // Correct
   useEffect(() => {
     document.title = `Count: ${count}`;
   }, [count]);
   ```

3. **Ref Access:**
   ```jsx
   // Wrong
   useEffect(() => {
     inputRef.current.focus();
   }); // Might try to focus before ref is attached

   // Correct
   useEffect(() => {
     if (inputRef.current) {
       inputRef.current.focus();
     }
   }, []);
   ```
