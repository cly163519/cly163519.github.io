## Python Data Science Stack: A Beginner's Guide

# Overview

When learning Machine Learning with Python, you'll encounter these essential tools:

| Tool | What it is | What it does |
|------|-----------|--------------|
| **Python** | Programming language | The foundation for writing code |
| **NumPy** | Python library | Fast numerical computations (arrays, matrices) |
| **Pandas** | Python library | Data manipulation (like Excel for Python) |
| **Scikit-learn** | Python library | Machine Learning algorithms |
| **Kernel** | Jupyter backend | The engine that runs your code |
| **LaTeX** | Typesetting system | For writing math formulas beautifully |

# How They Work Together

```
Python (base language)
  │
  ├── NumPy (numerical computing)
  │     └── Arrays, matrices, math operations
  │
  ├── Pandas (data handling)
  │     └── Read CSV, manipulate tables
  │
  └── Scikit-learn (machine learning)
        └── Classification, prediction, model training
```

# 1. Python

Python is the programming language itself - the foundation everything else is built on.

```python
# Basic Python
for i in range(5):
    print(i)
```

# 2. NumPy

NumPy provides fast array operations and mathematical functions.

```python
import numpy as np

# Create arrays
a = np.array([1, 2, 3, 4, 5])
b = np.ones((3, 4))          # 3x4 matrix of ones
c = np.random.rand(2, 3)     # Random 2x3 matrix

# Operations
print(a.shape)               # (5,)
print(a.max())               # 5
print(a.mean())              # 3.0

# Matrix multiplication
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])
print(A.dot(B))              # Matrix multiplication
```

# 3. Pandas

Pandas is for handling tabular data (rows and columns), like Excel.
```python
import pandas as pd

# Read CSV file
df = pd.read_csv('data.csv')

# View data
df.head()          # First 5 rows
df.tail(3)         # Last 3 rows
df.shape           # (rows, columns)
df.columns         # Column names
df.describe()      # Statistics summary

# Select data
df['column_name']           # Single column
df[['col1', 'col2']]        # Multiple columns
df.sample(5)                # Random 5 rows

# Separate features and labels
X = df.drop('target', axis=1)  # All columns except 'target'
y = df['target']               # Only 'target' column
```

# 4. Scikit-learn

Scikit-learn provides machine learning algorithms and tools.

```python
from sklearn import datasets
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

# Load built-in dataset
iris = datasets.load_iris()
X = iris.data      # Features
y = iris.target    # Labels

# Split data into training and testing sets
X_train, X_test, y_train, y_test = train_test_split(
    X, y, 
    test_size=0.2,      # 20% for testing
    random_state=42     # For reproducibility
)

# Standardize the data
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)
```

# 5. Kernel

Kernel is the "engine" behind Jupyter Notebook that actually runs your code.
```
Jupyter Notebook (what you see)
        │
        ▼
     Kernel (runs code behind the scenes)
        │
        ▼
     Output (results you see)
```

- When you click "Run", the code is sent to the kernel
- The kernel executes the code and returns the result
- Different kernels for different languages: Python, R, Julia, etc.
- If your code gets stuck, you can "Restart Kernel" to start fresh

Common kernel operations in Jupyter:
- **Restart Kernel**: Clear all variables, start over
- **Interrupt Kernel**: Stop running code
- **Change Kernel**: Switch to different Python version

# 6. LaTeX

LaTeX is a typesetting system for writing professional-looking math formulas.

In Jupyter Notebook, you can write LaTeX in Markdown cells:
```
# Inline math (single $)
The formula is $E = mc^2$

# Block math (double $$)
$$
\frac{a}{b} = \frac{c}{d}
$$
```

Common LaTeX examples:

| What you want | LaTeX code | Result |
|---------------|------------|--------|
| Fraction | `$\frac{a}{b}$` | a/b |
| Square root | `$\sqrt{x}$` | √x |
| Superscript | `$x^2$` | x² |
| Subscript | `$x_i$` | xᵢ |
| Sum | `$\sum_{i=1}^{n}$` | Σ |
| Greek letters | `$\alpha, \beta, \gamma$` | α, β, γ |

Example - Mean formula:
```
$$\bar{x} = \frac{1}{n}\sum_{i=1}^{n}x_i$$
```

# Common Workflow
```
1. Load data         → pd.read_csv() or datasets.load_iris()
2. Explore data      → df.head(), df.shape, df.describe()
3. Prepare data      → train_test_split()
4. Preprocess        → StandardScaler()
5. Train model       → model.fit()
6. Evaluate          → model.score()
```

# Quick Reference

# Import Statements
```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from sklearn import datasets
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
```

# NumPy vs Pandas

| Task | NumPy | Pandas |
|------|-------|--------|
| Data structure | ndarray | DataFrame / Series |
| Best for | Numerical computation | Tabular data |
| Column names | No | Yes |
| Data types | Single type | Mixed types |
| From sklearn | `iris.data` returns ndarray | `load_iris(as_frame=True)` returns DataFrame |

# Summary

- **Python**: The language you write in
- **NumPy**: Makes math fast
- **Pandas**: Makes data easy to work with
- **Scikit-learn**: Makes machine learning accessible
- **Kernel**: The engine that runs your code in Jupyter
- **LaTeX**: Makes math formulas look professional

They work together: Load data with Pandas → Process with NumPy → Train models with Scikit-learn!
