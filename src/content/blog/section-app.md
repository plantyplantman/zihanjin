---
author: Zihan Jin
pubDatetime: 2023-06-12T03:16:52.737Z
title: Building a Full-stack Legislation Chatbot Application
postSlug: section-app
featured: true
ogImage: https://user-images.githubusercontent.com/53733092/215771435-25408246-2309-4f8b-a781-1f3d93bdf0ec.png
tags:
  - release
  - ai
  - fullstack
  - law
description: The process of designing, building and deploying a full-stack legislation chatbot application
---

## Table of contents

## Introduction

Imagine if you could interact with an immense, complex body of textual data—like legislation—and it could not only understand your questions, but also respond accurately. Intriguing, right? It's this vision that I have embarked on turning into a reality with a project aimed at democratizing access to legislation.

While legislation is accessible online, its dense legal jargon often makes understanding it difficult. My project aims to remove this barrier, enabling anyone to swiftly query and comprehend active legislation.

Please note: This project is currently under active development, so substantial changes are to be expected in the near future.

[Visit the site here](selenium-nextjs.vercel.app)

## Technical Framework

To keep things manageable and minimize complexity in these early stages of development, I have designed the site with a serverless architecture.

#### Frontend

The frontend is developed using NextJS 13.4 with the App Router and Server Actions hosted on Vercel

#### Data storage and management

For data storage and management, I am using Postgres, hosted on Supabase. Google cloud storage provides content delivery network (CDN) capabilities. Currently, Pinecone is my vector store, but I'm in the process of migrating to a dockerised PSQL, coupled with PGVector.

#### Embeddings

In order to generate embeddings, I'm utilizing the [INSTRUCTOR Large model](https://instructor-embedding.github.io/). I have set up a FastAPI+Uvicorn server in a Docker container, which is hosted on Google Cloud Run for serverless inference.

#### Completion model

When it comes to completion models, none of the open-source options can quite match up to the performance of OpenAI's GPT family. As a result, I am using OpenAI's chat completion API for inference.

I am also keeping a close eye on the development of Microsoft's foundational model, [Orca](https://arxiv.org/pdf/2306.02707.pdf). Despite being less than 10% the size (13B), Orca outperforms GPT-3.5-turbo on a variety of tasks including LogiQA, a logical question answering dataset. Once it's released, I plan to explore its potential for integration into my system.

The journey towards AI-driven legislation comprehension is a thrilling one, and I am excited to be on this path. Now, let's delve into the details of how I've sourced and managed the vast amount of data that powers this project.

## Data collection

As of 12/06/2023, I have managed to scrape, index, and embed all active legislation from the Australian Capital Territory (ACT). All legislation was gathered from [AustLII](https://www.austlii.edu.au/) utilising [Selenium](https://pypi.org/project/selenium/) and the Chromium driver.

### Scraping

You can find the code I used for scraping [here](https://github.com/plantyplantman/legislation-scraping).
Initially, I leveraged selenium to concurrently traverse AUSTLII, aiming to download all text files for each piece of legislation. However, I soon recognized that this method was insufficient, as the downloaded text files couldn't be easily divided into semantically meaningful sections.

To overcome this, I opted for a different approach in my second attempt. Instead of downloading text files, I decided to directly scrape the HTML of each piece of legislation. This was feasible as AUSTLII organizes each section of the legislation on its own page, with an index page acting as the entry point.
![AustLII website structure](/assets/section-article/0.png)

#### The scaping process

Below are some screenshots to give you a visual idea of the scraping process:
![Scraping process 1](/assets/section-article/1.png)
![Scraping process 2](/assets/section-article/2.png)
![Scraping process 3](/assets/section-article/3.png)
![Scraping process 4](/assets/section-article/4.png)

#### Parsing and indexing

The scraping process resulted in individual folders for each piece of legislation. Within these folders are a series of HTML files representing each section of the legislation. The naming convention for these files follows the `{category}{order}{name of section}.html` format.

**The categories could be one of the following:**

1. INDEX
2. LONG TITLE
3. NOTES
4. SECT
5. SCHEDULE

I created a Python script to organize this data. The script navigates the directory tree of legislation and adds information to a JSON file based on the rules mentioned above. This file was later used to upsert the data to the PSQL database. To ensure consistency between the PSQL database and the Pinecone vector index, I assigned a UUID to each section.

Additionally, I created a Google Cloud Storage link for each section and included it in the JSON file. This JSON file was then used to insert the legislation, its metadata, its sections, and the sections' metadata into the PSQL database.

## Embedding

### Understanding Embeddings

Embedding is a powerful method to transform words, sentences, paragraphs, and even whole documents into n-dimensional vectors. This transformation allows us to perform vector space computations to identify which pieces of text are similar to the input. Typically, these computations use algorithms like K-nearest-neighbours, cosine similarity, or dot product.

![Visualisation of embedded books in vector space](https://miro.medium.com/v2/resize:fit:1400/1*zhlXuzV2kI2V2qJ5M3uPPg.gif)
^ an interactive exploration of book embeddings using [projector](https://projector.tensorflow.org/). Credit: [Will Koehrsen](https://towardsdatascience.com/neural-network-embeddings-explained-4d028e6f0526)

The technique of embedding isn't novel; it has been employed for decades to search for and retrieve semantically similar content. However, earlier efforts to represent words as vectors often fell short due to the complex nature of human language and struggled to capture higher-level conceptual meaning.

The arrival of Transformer-based models like BERT or GPT, which are trained on large volumes of text, has significantly changed the field. These models can represent the complex relationships between words, sentences, paragraphs and more as dense vectors. We can now efficiently find conceptually similar texts based on input.

There are several commercially available embedding models, such as OpenAI's ADA-002 or Cohere's multilingual embedding model. On the other hand, open-source embedding models have gained popularity due to their smaller size (100m-1.5b), making them suitable for consumer hardware CPUs. One standout open-source model is the [INSTRUCTOR](https://instructor-embedding.github.io/) model from the University of Hong Kong, University of Washington, Meta and the Allen Institute. Its unique feature is the ability to use instruction+text pairs to guide the model in capturing the desired aspects of the text.

### The Embedding process

For this project, I utilized the INSTRUCTOR model to transform each section of text into a 768-dimensional vector. Due to the model's maximum context length of 512 tokens (approximately 2048 characters), some sections of legislation were too long to feed directly into the model. In these cases, I split the section into equal-length chunks with an overlap of 200 characters to ensure no loss of crucial information.

Once each chunk or section was embedded, the resulting vector was upserted to Pinecone. You can find the embedding script [here](https://github.com/plantyplantman/legislation-scraping/blob/57b176b98da67baf1dfe45fe9dacfb16dd87f24b/scripts/embed.py).

However, Pinecone has proven not to be the optimal vector store for this project due to its limited functionality in the free tier and its slow performance with high cardinality metadata like UUIDs. As a result, I'm migrating to PSQL with the PGVector extension, which provides support for high dimensionality vectors and maintains the benefits of PSQL. Advantages include foreign key support for vector tables, efficient indexing for high cardinality values, and, most importantly, it's free.

### Deploying INSTRUCTOR

To query a vector store, it's crucial to use vectors of the same dimensionality. Additionally, to ensure the best results, the same embedding model should be used for your queries and your vector store, providing consistent semantic representations.

To adhere to the serverless design of this project, I decided to host the embedding model on Google Cloud Run. I created a minimal FastAPI+Uvicorn server within a Docker container, which provides an API endpoint to transform user queries into 768-dimensional vectors for retrieval. You can find the API [here](https://github.com/plantyplantman/instructor-embed-api).

If you want to test the API, use the following code:

```sh
curl -X POST "https://default-service-ssgtgd52nq-ts.a.run.app/embed" \
     -H "Content-Type: application/json" \
     -d '{
         "texts": ["your-texts-here"],
         "instruction": "your-instruction-here"
     }'
```

Keep in mind that your first request may encounter a cold start, potentially requiring 30-45s to boot up. However, subsequent inference should be significantly faster.

## Frontend & Frontend Server

[Code available here](https://github.com/plantyplantman/selenium-nextjs/tree/main).

### How it works

Operational Overview
Upon initial site load, the client sends a request to the database via a server action to fetch the name and ID of all legislations. When a piece of legislation is selected by the user, the client router navigates to `/legislation/[legislationID]`. This dynamic route includes an asynchronous server component that fetches the `index_url` and `combined_url` of the chosen legislation from the database. These URLs point to HTML documents hosted on Google Cloud Storage, which are then fetched and displayed on the page. Concurrently, if the user is logged in, previous messages are fetched from the database and displayed.

When a user sends a message, the chat history and legislation data are sent to the endpoint `/API/message`. This endpoint then sends the latest user message to the embedding endpoint and awaits the resulting vector. This vector is sent to Pinecone for a cosine similarity search over vectors with a matching legislationID. The texts of the top K vectors are then combined with the chat history and formatted with a predfined system prompt to create the system prompt for the current message. The system prompt and chat history are then sent to the OpenAI completion API, and the response is streamed back to the client to be displayed in the chat box. This retrieval augmented question answering method is designed to provide factual responses and reduce the chance of "hallucinations".
![What happens on mount](/assets/section-article/5.png)

![What happens on message send](/assets/section-article/6.png)

### Technologies

#### NextJS 13.4

I utilised NextJS 13.4 for the frontend and frontend server due to its growing popularity for building serverless full-stack React applications. It provides an exceptional developer experience with server actions that streamline the definition of server-only functions. With asynchronous React server components, coding in React becomes significantly more enjoyable.

NextJS with server actions provides a fantastic developer experience allowing you to easily define functions that are to only run on the server and call them from client components. This reduces the need to create convoluted and mostly unnecessary API routes for trivial things like fetching data from a database. These server actions combined with asynchronous React server components makes writing React a million times more tolerable.

There was one glaring issue that I did not realise until it was too late...

One major challenge I encountered was working with external HTML sources in React. Given the legislative data I scraped was in HTML format, the only method I could use to display it was `dangerouslySetInnerHTML`, which carries significant security risks.

In addition, it is very difficult to style and edit individual elements within the dangerous HTML. As a workaround, I defined some styles in a string and before rendering the HTML, I appended the styles to the HTML document. This method is not at all elegant and is quite ugly, but given the circumstance, I think it works well enough.

In hindsight, using a tool like Svelte or Solid, which support displaying HTML in components, might have been a better choice.

#### Supabase

Supabase is an open-source Software-as-a-Service (SaaS) platform that simplifies full-stack development. It offers a range of services, such as database hosting, serverless actions, authentication, and blob storage. I initially used Vercel's Postgres service for database hosting but switched to Supabase due to its generous free tier and user-friendly interface. As the project evolved, I migrated from NextAuth to Supabase for user authentication, further simplifying the process.

#### OpenAI & Pinecone SDK's (lack thereof)

For this project, I chose not to use the SDKs provided by OpenAI and Pinecone, as I felt they added an unnecessary layer of complexity. Unless you need to, you can often make do with issuing HTTP requests to their APIs.

#### Tailwind CSS

Tailwind CSS greatly simplifies working with CSS. Notably, the command bar interface of the site was inspired by the command bar in Tailwind's documentation.

#### Prisma

At the project's outset, I used Prisma to streamline development. However, after understanding its workings, I realized that it's not entirely suitable for serverless deployment due to a large Rust binary that serves as a translation layer for queries. This layer can lead to multiple database requests for more complex queries, slowing down performance and potentially increasing costs. As a result, I transitioned to using Supabase's Javascript client for all database queries.

## Conclusion

In summary, the journey of building a legal chatbot that employs the latest advances in natural language processing, embedding, and serverless infrastructure was both challenging and rewarding. This project allowed me to explore different technologies, like NextJS, Supabase, OpenAI, Pinecone, and others, in a practical and meaningful way.

Despite encountering several roadblocks along the way, such as issues with external HTML sources and the suitability of certain tools for serverless deployment, these challenges ultimately deepened my understanding and sharpened my problem-solving skills. In retrospect, some choices might have been made differently, like the option of using a platform with built-in support for displaying HTML in components.

As with any tech project, continuous refinement is key. Currently, I'm in the process of transitioning away from Pinecone as the vector store and aiming for more suitable solutions like PSQL with the PGVector extension. It's a reminder that the tech landscape is dynamic and solutions that worked yesterday may not be the best for tomorrow.

On the whole, this project serves as a testament to the potential of AI in making intricate legal texts accessible to a broader audience. By combining powerful technologies like transformer-based language models and serverless infrastructure, we can democratize access to legal information and hopefully foster better understanding and engagement with the laws that govern our societies.

The journey does not end here. With continuous improvements and updates, this chatbot will strive to better serve its users, showcasing the enormous potential of AI and its capability to revolutionize various sectors, including the legal domain. The future promises even more exciting developments, and I look forward to being part of that journey.
