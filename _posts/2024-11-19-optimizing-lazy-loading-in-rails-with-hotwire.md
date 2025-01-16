---
layout: post
title: "Optimizing lazy loading in Rails with Hotwire"
author: "Alexis Crozier"
categories: facts
tags: [hotwire]
image: posts/lazyload-hotwire.png
---

## Table of Contents

1. What is lazy loading?
2. Why implement lazy loading? 
3. How to implement lazy loading (without Hotwire)
4. Lazy loading with Hotwire 
5. Conclusion

## 1. What is Lazy Loading?

Lazy loading is a data-loading technique that contrasts with eager loading. 
While eager loading involves loading all data at once, lazy loading works on the opposite principle: data is only loaded 
when it is needed.

This approach is widely used to optimize loading times and application 
performance. 
There are various forms of lazy loading, such as:

- Infinite scroll, where content loads progressively as the user scrolls,
- Delayed loading of images, which only happens when they become visible in the user's viewport.

## 2. Why Implement Lazy Loading?

Implementing lazy loading provides several benefits for user experience and server performance:

- Improved User Experience: By deferring the loading of certain data, the initial page load time is optimized, ensuring smoother and more enjoyable navigation.
- Reduced Server Resource Usage: Lazy loading minimizes unnecessary requests by loading only the required data. This technique avoids processing potentially redundant data, reducing computation times and limiting requests to the essentials.

In summary, lazy loading helps prevent resource overload by limiting expensive operations, 
thereby enhancing application performance and responsiveness.

## 3. How to Implement Lazy Loading (without Hotwire)
![js header]("{{ '/assets/img/posts/js-header.png' | relative_url }}")

Lazy loading can be implemented in JavaScript using techniques like IntersectionObserver to detect when an element becomes visible in the viewport, 
and the fetch API to retrieve data as needed. 

However, these traditional approaches quickly show limitations in dynamic applications where server interactions are frequent.

### Lazy Loading with IntersectionObserver

The IntersectionObserver API in JavaScript is a common method to detect when an element enters the user's viewport. 
It triggers actions (like loading images or components) as soon as an element becomes visible, optimizing the visual rendering.

However, IntersectionObserver is not designed to directly handle dynamic server data. 
Its role is limited to observing an element's visibility, and it needs to be combined with the fetch API and Promises to dynamically load server-based content, 
such as article lists or more complex components.

### Fetch API and Promises: A Solution with Some Drawbacks

The fetch API in JavaScript allows you to make HTTP requests to the server to retrieve data as needed, 
typically in response to user actions. 
This approach uses Promises to handle asynchronous requests without blocking the user interface.

Basic Workflow of the Fetch API:

- Use fetch() to make an HTTP request to the server, receiving a response as a Promise. Once the Promise resolves, the content can be accessed and injected into the page.
- This method is effective for on-demand content loading but requires careful synchronization to integrate the retrieved data properly into the application.

Challenges with Using Fetch and IntersectionObserver:

1. Visibility and Data Synchronization: The request should only trigger when the element is visible, and the content must load correctly. This requires maintaining precise state management between element visibility and server response, which can become complex if multiple elements need cascading loading.
2. Error Handling: Each fetch request must include error management to avoid partially loaded content or uncontrolled error messages for the user. Issues like network errors or long response times can lead to a poor user experience.
3. Partial Content Refresh: In applications where elements require dynamic reloading, managing these refreshes can be cumbersome. Properly structuring the code is essential to avoid duplicates, manage latency, and ensure components don’t reload multiple times or incorrectly.

These constraints can quickly make the code complex to maintain, especially in projects involving frequent user interactions and server calls.

Example: Lazy Loading a Modal with IntersectionObserver and Fetch API

```javascript
document.addEventListener("DOMContentLoaded", function () {
  const modal = document.getElementById("detailsModal");
  const modalContent = document.getElementById("modalContent");
  const loadModalButton = document.getElementById("loadModalButton");

  let observer;

  // Load data with fetch()
  async function loadModalContent() {
    try {
      modalContent.innerHTML = "<p>Loading...</p>"; // Display a spinner
      const response = await fetch('/path/to/api/articles');

      if (!response.ok) {
        throw new Error("Error fetching data");
      }

      const data = await response.json();
      modalContent.innerHTML = `<h2>${data.title}</h2><p>${data.content}</p>`;
    } catch (error) {
      console.error(error);
      modalContent.innerHTML = "<p>Error fetching data.</p>";
    }
  }

  // Function to observe when the button enters the viewport
  function createObserver() {
    observer = new IntersectionObserver((entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          loadModalContent();
          modal.style.display = "block";
          observer.disconnect(); // Avoid multiple calls
        }
      });
    });
    observer.observe(modalContent);
  }

  createObserver(); // Initiate the observer when content is visible
});
```

## 4. Lazy Loading with Hotwire
### Loading Data into a Modal

As seen earlier, it’s possible to use the fetch API in JavaScript to dynamically load data into a modal. However, this requires significant JavaScript code to manage Promises and ensure data is fetched and injected correctly.

With **Hotwire**,
this logic is simplified by using turbo_frame_tag,
which makes it easy to implement lazy loading for modal content. 
**Hotwire** automatically handles data loading via HTTP requests without the need for extra JavaScript.

### Steps to Implement Lazy Loading with Hotwire
#### Backend Setup

- Define a route for the modal content. This route will be passed as the src attribute in the turbo_frame_tag.

```ruby
# config/routes.rb
get :modal_content, to: 'controller#modal_content', as: 'object_details'
```

- Create the modal_content action in the relevant controller. 
This action prepares the required data and renders the partial to be displayed in the modal.

```ruby
# app/controllers/your_controller.rb
def modal_content
  render partial: 'path/to/your/partial'
end
```

- That’s it for the backend setup ! :rocket:

#### Frontend Setup
To implement lazy loading with Hotwire on the frontend, 
we will use turbo_frame_tag in our views. 
This will allow us to dynamically load the modal's content at the moment it is opened, without requiring additional JavaScript to handle asynchronous requests.

In the main view, we start by creating a link or button that will trigger the modal's opening. 

Here is an example:

```erb
# In the view where we'll display the modal

# Button to display the modal
<%= link_to '#',
            class: 'btn btn-bg-red text-light my-1 rounded-circle d-flex justify-content-center align-items-center',
            type: 'button',
            style: "width: 50px; height: 50px;",
            data: {
              bs_toggle: 'modal',
              bs_target: "#detailsObject#{@object.id}",
            } do %>
  <%= bootstrap_icon "search", width: 25 %>
<% end %>

# The modal 
<div class="modal fade" id="detailsObject<%= @object.id %>" tabindex="-1" aria-labelledby="detailsObjectLabel" aria-hidden="true">
  <div class="modal-dialog modal-lg modal-dialog-centered">
    <div class="modal-content">
      <!-- Here, we'll add the turbo_frame_tag -->
    </div>
  </div>
</div>
```

-  Integration of turbo_frame_tag in the modal

The core of the lazy loading implementation is found in the modal's content. Inside it, we insert a turbo_frame_tag to dynamically load the content via an HTTP request as soon as the modal becomes visible.

```erb
<div class="modal fade" id="detailsObject<%= @object.id %>" tabindex="-1" aria-labelledby="detailsObjectLabel" aria-hidden="true">
  <div class="modal-dialog modal-lg modal-dialog-centered">
    <div class="modal-content">

      <!-- Hotwire -->
      <%= turbo_frame_tag 'modal',
                          src: modal_content_path(id: @object.id),
                          loading: "lazy" do %>
        <!-- Spinner -->
        <div class="d-flex justify-content-center align-items-center my-2">
          <div class="spinner-border mb-5 mt-5" style="width: 5rem; height: 5rem;" role="status"></div>
        </div>
      <% end %>
    </div>
  </div>
</div>
```

**Explanation of the turbo frame elements:**

- `turbo_frame_tag "modal"` : creates a turbo-frame element with the ID modal. This frame acts as a container in which Hotwire will load the content.
- `src` : specifies the route to the controller action we defined in the backend section, allowing access to the necessary data for the modal.
- `loading: "lazy"` : indicates that this content should be loaded only when it becomes necessary.
- **Spinner**: we add a loading indicator inside the turbo_frame_tag, which is displayed while the content is loading. This spinner is automatically replaced by the content once the request is completed.

**Creating the partial for the modal content**

Finally, we define the partial rendered by our controller action. 
This partial contains the dynamic content of the modal and is wrapped in a turbo_frame_tag with the same ID.

This ensures that the content loads into the correct container in our view.

```erb
<%= turbo_frame_tag 'modal' do %>
  <!-- Modal content here -->
<% end %>
```

![gif]("{{ '/assets/img/posts/awesome.gif' | relative_url }}")

<hr/>

## Conclusion

In conclusion, 
Hotwire allows us to implement lazy loading in a simpler way than using 
IntersectionObserver and the Fetch() API. 
Furthermore, this implementation allows us to better respect the MVC pattern of Rails and 
its conventions by adhering to the strategy:

Back-End manages the construction of data as well as any server requests.

Front-end manages the display and layout of data coming from our Back-end.

### Key points to remember

- Lazy loading Saving computational resources and server requests Improved user experience
- Hotwire Simplification of lazy loading implementation Robust implementation Easy to maintain Respects the RAILS pattern
