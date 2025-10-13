---
layout: post
title:  "Spec driven development: The AI way"
date:   2025-10-10 00:00:00 +1300
categories: AI
author: Jose Rodriguez
---

Have you ever built a feature, only to realize it was not what the project manager wanted? Or have you fixed a bug, and another one appeared? These are common problems in software development. Often, they happen because we rush to write code without a clear plan.

Today, we will explore a powerful method to solve this: **Spec-Driven Development (SDD)**. And we will see how to use it with an AI coding assistant like Claude to make it even faster and more effective.

### What is Spec-Driven Development?

Simply put, Spec-Driven Development is the practice of **writing a detailed plan before you write any code.**

This plan is called a "specification" or "spec." It is a blueprint for your feature. It describes exactly what the code should do, how it should behave, and how to know if it is working correctly. It is much more than a simple comment; it is a formal agreement on the work to be done.

A good spec usually includes:

*   **Goal/User Story:** *Why* are we building this? What problem does it solve for the user?
*   **Features:** *What* will the code do? What are its inputs and outputs?
*   **Technical Plan:** A high-level idea of *how* we will build it. What technologies or functions will we use?
*   **Acceptance Criteria:** How do we prove it works? These are simple "pass" or "fail" statements that will become our tests.

### The AI Superpower: From Spec to Code

This is where an AI assistant becomes your partner. Instead of you writing all the code, you can give the spec to the AI and ask it to generate the code for you.

**You write the plan, the AI writes the code.** Your job then becomes to check, test, and refine the AI's output.

### A Complex Example: The Smart User Profile Card

Let's imagine we need to build a user profile card for our web application. This is not a simple card. It must show different information depending on who is looking at it.

*   A **regular user** should only see the basic information: name, picture, and job title.
*   An **admin** should see everything: basic information plus sensitive details like email and phone number.

This is a perfect case for SDD because there is business logic involved. A misunderstanding here could lead to showing private information to the wrong people!

#### Step 1: Write the Spec

First, we create a detailed spec. This is what we will give to our AI assistant.

```markdown
**Component Name:** `UserProfileCard`

**Goal:** To display a user's profile information. The information shown must change based on the viewer's role (admin or user). This is to protect sensitive user data.

**Technology:** React.js

**Props (Inputs):**
- `user`: An object with the profile information.
  - `name` (string)
  - `imageUrl` (string)
  - `title` (string)
  - `email` (string)
  - `phone` (string)
- `viewer`: An object representing the person looking at the profile.
  - `role` (string: 'admin' or 'user')

**Features:**
1.  The component must display the user's `name`, `imageUrl`, and `title` to everyone.
2.  If the `viewer.role` is `'admin'`, the component must ALSO display the user's `email` and `phone`.
3.  If the `viewer.role` is not `'admin'`, the `email` and `phone` must NOT be displayed.
4.  The component should have clean and modern styling.

**Acceptance Criteria:**
1.  **Given** a viewer with the role `'user'`, **when** the component renders, **then** only the name, image, and title are visible.
2.  **Given** a viewer with the role `'admin'`, **when** the component renders, **then** the name, image, title, email, and phone are all visible.
```

This spec is clear, detailed, and leaves no room for confusion.

#### Step 2: Give the Spec to the AI

Now, we copy this spec and give it to our AI assistant. We can say:

*"Hello Claude, please generate a React component based on the following spec."*

#### Step 3: Get the Code from the AI

The AI will analyze the spec and generate the code. Here is what we might get back:

```jsx
// UserProfileCard.js
import React from 'react';
import './UserProfileCard.css';

const UserProfileCard = ({ user, viewer }) => {
  if (!user || !viewer) {
    return <div>Loading...</div>;
  }

  const isAdmin = viewer.role === 'admin';

  return (
    <div className="profile-card">
      <img src={user.imageUrl} alt={`${user.name}'s profile`} className="profile-image" />
      <h2 className="profile-name">{user.name}</h2>
      <p className="profile-title">{user.title}</p>
      
      {isAdmin && (
        <div className="admin-details">
          <hr />
          <p><strong>Email:</strong> {user.email}</p>
          <p><strong>Phone:</strong> {user.phone}</p>
        </div>
      )}
    </div>
  );
};

export default UserProfileCard;
```

And the CSS for styling:

```css
/* UserProfileCard.css */
.profile-card {
  font-family: sans-serif;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  padding: 24px;
  max-width: 300px;
  text-align: center;
  background-color: white;
}

.profile-image {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  object-fit: cover;
  border: 3px solid #f0f0f0;
}

.profile-name {
  margin: 16px 0 4px;
  color: #333;
}

.profile-title {
  margin: 0;
  color: #777;
  font-weight: 500;
}

.admin-details {
  margin-top: 16px;
  text-align: left;
  color: #555;
}

.admin-details hr {
  border: none;
  border-top: 1px solid #eee;
  margin: 16px 0;
}
```

Look at this! The AI understood the logic. It created a component that conditionally renders the admin details using `isAdmin && (...)`. This is a direct translation of our spec into code.

#### Step 4: Test the Code Against the Spec

Now, we must verify the code. Our spec's "Acceptance Criteria" become our test plan.

We can create a test file to automate this. Here is an example using Jest and React Testing Library:

```jsx
// UserProfileCard.test.js
import React from 'react';
import { render, screen } from '@testing-library/react';
import UserProfileCard from './UserProfileCard';

// Mock data for our tests
const mockUser = {
  name: 'Alex Doe',
  imageUrl: 'https://example.com/alex.jpg',
  title: 'Software Engineer',
  email: 'alex.doe@example.com',
  phone: '123-456-7890',
};

const regularViewer = { role: 'user' };
const adminViewer = { role: 'admin' };

test('Acceptance Criterion 1: shows only basic info for a regular user', () => {
  render(<UserProfileCard user={mockUser} viewer={regularViewer} />);

  // Check that basic info is visible
  expect(screen.getByText('Alex Doe')).toBeInTheDocument();
  expect(screen.getByText('Software Engineer')).toBeInTheDocument();
  expect(screen.getByAltText("Alex Doe's profile")).toBeInTheDocument();

  // Check that admin info is NOT visible
  expect(screen.queryByText(/alex.doe@example.com/i)).not.toBeInTheDocument();
  expect(screen.queryByText(/123-456-7890/i)).not.toBeInTheDocument();
});

test('Acceptance Criterion 2: shows all info for an admin user', () => {
  render(<UserProfileCard user={mockUser} viewer={adminViewer} />);

  // Check that basic info is visible
  expect(screen.getByText('Alex Doe')).toBeInTheDocument();
  expect(screen.getByText('Software Engineer')).toBeInTheDocument();

  // Check that admin info is ALSO visible
  expect(screen.getByText(/alex.doe@example.com/i)).toBeInTheDocument();
  expect(screen.getByText(/123-456-7890/i)).toBeInTheDocument();
});
```

When we run these tests, they will pass. We have now *proven* that our code meets the requirements of the spec.

### Why This is a Better Way to Work

1.  **Clarity First.** By writing a spec, you force yourself to think through all the details before coding. This reduces mistakes and misunderstandings.
2.  **Faster Development.** It may seem slower to plan first, but it is not. The AI generates the first version of the code in seconds. You spend your time thinking and reviewing, not typing boilerplate code.
3.  **Fewer Bugs.** The spec clearly defines the correct behavior. This makes it easy to write accurate tests and catch bugs before they reach users.
4.  **Better Teamwork.** A spec is a perfect document to share with project managers, designers, and other engineers. Everyone can agree on the plan before the work starts.

### Conclusion

Spec-Driven Development is a powerful technique for building reliable software. When you combine it with an AI coding assistant, it becomes a true superpower. You provide the expert direction, and the AI handles the repetitive work of writing the code.

The next time you start a new feature, don't open your code editor first. Open a text file, write a clear spec, and let your new AI partner help you build it right, the first time.
