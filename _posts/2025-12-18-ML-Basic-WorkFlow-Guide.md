---
layout: post
title: "The Complete Machine Learning Workflow: A Step-by-Step Guide"
date: 2024-12-18
categories: [Machine Learning, Python, Scikit-learn]
tags: [KNN, Decision Tree, Classification, Tutorial]
---

# The Complete Machine Learning Workflow: A Step-by-Step Guide

After working through multiple machine learning notebooks, I've identified a **consistent pattern** that appears in virtually every ML classification project. This guide breaks down the universal workflow into 10 clear steps that you can follow for any supervised learning task.

## The Universal ML Pipeline

```
┌─────────────────────────────────────────────────┐
│  Step 1:  Import Libraries                      │
│  Step 2:  Load Data                             │
│  Step 3:  Split Data (Train / Test)             │
│  Step 4:  Standardize (Required for KNN)        │
│  Step 5:  Create Model                          │
│  Step 6:  Train Model                           │
│  Step 7:  Make Predictions                      │
│  Step 8:  Evaluate Performance                  │
│  Step 9:  Visualize Results (Optional)          │
│  Step 10: Tune Hyperparameters                  │
└─────────────────────────────────────────────────┘
```

---

## Step 1: Import Libraries

Every ML project starts with importing the necessary libraries.

```python
# Basic libraries
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

# Datasets
from sklearn import datasets
from sklearn.datasets import load_iris, load_wine, load_digits

# Data splitting
from sklearn.model_selection import train_test_split

# Preprocessing
from sklearn.preprocessing import StandardScaler

# Models
from sklearn.neighbors import KNeighborsClassifier
from sklearn.tree import DecisionTreeClassifier, export_text, plot_tree

# Evaluation metrics
from sklearn.metrics import accuracy_score
from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay
from sklearn.metrics import classification_report
```

---

## Step 2: Load Data

There are two common ways to load data in scikit-learn projects.

### Option A: Built-in Datasets

```python
# Load sklearn built-in dataset
iris = datasets.load_iris()

X = iris.data                    # Features (150, 4)
y = iris.target                  # Labels (150,)
feature_names = iris.feature_names   # ['sepal length', 'sepal width', ...]
class_names = iris.target_names      # ['setosa', 'versicolor', 'virginica']
```

### Option B: CSV Files

```python
# Load from CSV file
df = pd.read_csv('./data.csv')

X = df.drop('Class', axis=1)         # All columns except 'Class'
y = df['Class']                       # Only the 'Class' column
feature_names = list(X.columns)       # Get feature names from DataFrame
class_names = ['Class_0', 'Class_1']  # Define manually
```

---

## Step 3: Split Data

Splitting data ensures your model is tested on unseen data.

### Basic Split (Train + Test)

```python
X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,       # 20% for testing, 80% for training
    random_state=42,     # Reproducible results
    stratify=y           # Maintain class proportions
)
```

### Three-Way Split (Train + Validation + Test)

Use this when tuning hyperparameters:

```python
# First split: 60% train, 40% temp
X_train, X_temp, y_train, y_temp = train_test_split(
    X, y, test_size=0.4, random_state=42, stratify=y
)

# Second split: 20% validation, 20% test
X_val, X_test, y_val, y_test = train_test_split(
    X_temp, y_temp, test_size=0.5, random_state=42, stratify=y_temp
)
```

**Why three splits?**

| Set | Purpose |
|-----|---------|
| Training | Learn patterns |
| Validation | Tune hyperparameters |
| Test | Final evaluation (never peek!) |

---

## Step 4: Standardize Features

Standardization transforms features to have mean=0 and std=1.

```python
scaler = StandardScaler()
X_train = scaler.fit_transform(X_train)  # Fit on training data
X_test = scaler.transform(X_test)        # Transform test data
```

**Critical Rule:**
- `fit_transform()` on training data (learns parameters)
- `transform()` on test data (uses learned parameters)

**When to Standardize:**

| Algorithm | Standardization |
|-----------|-----------------|
| KNN | **Required** (distance-based) |
| Decision Tree | Not required (rule-based) |
| SVM | Required |
| Neural Networks | Required |

---

## Step 5: Create Model

### K-Nearest Neighbors (KNN)

```python
model = KNeighborsClassifier(n_neighbors=5)
```

Key parameter: `n_neighbors` (K) - number of neighbors to consider

### Decision Tree

```python
model = DecisionTreeClassifier(
    criterion="entropy",   # Use information gain
    max_depth=3,           # Limit tree depth
    random_state=42
)
```

Key parameters:
- `criterion`: "entropy" (information gain) or "gini"
- `max_depth`: Maximum tree depth (prevents overfitting)

---

## Step 6: Train Model

Training is always the same, regardless of algorithm:

```python
model.fit(X_train, y_train)
```

This single line does all the learning!

---

## Step 7: Make Predictions

```python
y_pred = model.predict(X_test)
```

---

## Step 8: Evaluate Performance

### Accuracy Score

```python
accuracy = accuracy_score(y_test, y_pred)
print(f'Accuracy: {accuracy:.2%}')
```

### Classification Report

```python
print(classification_report(y_test, y_pred, target_names=class_names))
```

Output:
```
              precision    recall  f1-score   support
      setosa       1.00      1.00      1.00        10
  versicolor       0.90      0.90      0.90        10
   virginica       0.90      0.90      0.90        10
    accuracy                           0.93        30
```

### Confusion Matrix

```python
ConfusionMatrixDisplay.from_estimator(
    model,
    X_test,
    y_test,
    display_labels=class_names,
    cmap=plt.cm.Blues,
    normalize=None  # Use "true" for percentages
)
plt.title("Confusion Matrix")
plt.show()
```

**Reading a Confusion Matrix:**
- Diagonal = Correct predictions
- Off-diagonal = Errors

---

## Step 9: Visualize (Decision Tree)

### Text Representation

```python
tree_rules = export_text(model, feature_names=list(feature_names))
print(tree_rules)
```

Output:
```
|--- petal length <= 2.45
|   |--- class: setosa
|--- petal length > 2.45
|   |--- petal width <= 1.75
|   |   |--- class: versicolor
|   |--- petal width > 1.75
|   |   |--- class: virginica
```

### Graphical Tree

```python
plt.figure(figsize=(15, 10))
plot_tree(
    model,
    feature_names=list(feature_names),
    class_names=list(class_names),
    filled=True
)
plt.show()
```

---

## Step 10: Tune Hyperparameters

Use a validation set to find the best parameters.

### Tuning K for KNN

```python
k_values = range(1, 21)
accuracies = []

for k in k_values:
    model = KNeighborsClassifier(n_neighbors=k)
    model.fit(X_train, y_train)
    y_val_pred = model.predict(X_val)
    acc = accuracy_score(y_val, y_val_pred)
    accuracies.append(acc)

# Plot K vs Accuracy
plt.plot(k_values, accuracies, marker='o')
plt.xlabel("K (Number of Neighbors)")
plt.ylabel("Validation Accuracy")
plt.title("Finding the Best K")
plt.grid(True)
plt.show()

# Find best K
best_k = k_values[accuracies.index(max(accuracies))]
print(f"Best K: {best_k}")
```

### Tuning max_depth for Decision Tree

```python
depth_values = range(1, 15)
accuracies = []

for depth in depth_values:
    model = DecisionTreeClassifier(criterion="entropy", max_depth=depth)
    model.fit(X_train, y_train)
    y_val_pred = model.predict(X_val)
    acc = accuracy_score(y_val, y_val_pred)
    accuracies.append(acc)

# Plot and find best depth
plt.plot(depth_values, accuracies, marker='o')
plt.xlabel("Max Depth")
plt.ylabel("Validation Accuracy")
plt.title("Finding the Best Max Depth")
plt.grid(True)
plt.show()
```

---

## Complete Template

Copy this template for any classification project:

```python
# ==================== STEP 1: IMPORTS ====================
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.neighbors import KNeighborsClassifier
from sklearn.metrics import accuracy_score, ConfusionMatrixDisplay

# ==================== STEP 2: LOAD DATA ====================
iris = load_iris()
X, y = iris.data, iris.target
feature_names = iris.feature_names
class_names = iris.target_names

# ==================== STEP 3: SPLIT DATA ====================
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# ==================== STEP 4: STANDARDIZE ====================
scaler = StandardScaler()
X_train = scaler.fit_transform(X_train)
X_test = scaler.transform(X_test)

# ==================== STEP 5: CREATE MODEL ====================
model = KNeighborsClassifier(n_neighbors=5)

# ==================== STEP 6: TRAIN ====================
model.fit(X_train, y_train)

# ==================== STEP 7: PREDICT ====================
y_pred = model.predict(X_test)

# ==================== STEP 8: EVALUATE ====================
accuracy = accuracy_score(y_test, y_pred)
print(f'Accuracy: {accuracy:.2%}')

ConfusionMatrixDisplay.from_estimator(
    model, X_test, y_test,
    display_labels=class_names,
    cmap=plt.cm.Blues
)
plt.title("Confusion Matrix")
plt.show()
```

---

## Quick Reference: KNN vs Decision Tree

| Aspect | KNN | Decision Tree |
|--------|-----|---------------|
| Key Parameter | `n_neighbors` (K) | `max_depth` |
| Standardization | **Required** | Not required |
| Interpretability | Low | High (visual rules) |
| Speed (prediction) | Slow (computes distances) | Fast |
| Handles outliers | Sensitive | Robust |

---

## Summary

The ML workflow can be summarized in one line:

```
Load → Split → Scale → Create → Fit → Predict → Evaluate
```

**Key Takeaways:**

1. **Always split data** before any preprocessing
2. **fit_transform on train**, transform on test
3. **Standardize for distance-based algorithms** (KNN, SVM)
4. **Use validation set for tuning**, test set only for final evaluation
5. **Same workflow, different models** - just swap Step 5!

---

* Written assisted with AI(Claude)
