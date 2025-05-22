# Snowflake Dev Day 2025

## Building Agentic AI Applications In Snowflake

## Overview

In this hands-on lab, you’ll learn how to build a Data Agent using Snowflake Cortex AI that can intelligently respond to questions by reasoning over both structured and unstructured data.

We’ll use a custom dataset focused on bikes and skis. This dataset is intentionally artificial, ensuring that no external LLM has prior knowledge of it. This gives us a clean and controlled environment to test and evaluate our data agent. By the end of the session, you’ll have a working AI-powered agent capable of understanding and retrieving insights across diverse data types — all securely within Snowflake.

### What You Will Learn

- How to setup your environment using Git integration and Snowflake Notebooks 
- How to work with semantic models and setup Cortex Analyst for structured data
- How to setup Cortext Search for unstructured data like PDFs and images
- How to use Cortex Agents REST API that uses these tools in a Streamlit application

### What You Will Need

* Use the account provided for you during the hands-on session, OR
* A [Snowflake Trial Account](https://signup.snowflake.com/)

## Step 1: Setup Git Integration 

Open a Worksheet, copy/paste the following code and execute all statements from top to bottom. This will set up the Git repository and will copy everything you will be using during the lab.

``` sql
CREATE or replace DATABASE DASH_CORTEX_AGENTS_SUMMIT;

CREATE OR REPLACE API INTEGRATION git_api_integration
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/Snowflake-Labs/')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY git_repo
    api_integration = git_api_integration
    origin = 'https://github.com/Snowflake-Labs/sfguide-build-data-agents-using-snowflake-cortex-ai';

-- Make sure we get the latest files
ALTER GIT REPOSITORY git_repo FETCH;

-- Setup stage for  Docs
create or replace stage docs ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = ( ENABLE = true );

-- Copy the docs for bikes
COPY FILES
    INTO @docs/
    FROM @DASH_CORTEX_AGENTS_SUMMIT.PUBLIC.git_repo/branches/main/docs/;

ALTER STAGE docs REFRESH;
```

## Step 2: Setup Unstructured Data Tools

We are going to be using a Snowflake Notebook to set up the tools that will be used by the Snowflake Cortex Agents API. Open the Notebook as shown below.

Select the Notebook that you have already available in your Snowflake account:

![image](img/1_create_notebook.png)

Give the notebook a name and select other options as shown below.

![image](img/2_run_on_wh.png)

Let's run through the cells in the Notebook. Here are some details to keep in mind.

Thanks to the GIT integration done in the previous step, the PDFs and image files we are going to be using have already been copied into your Snowflake account. We are using two sets of documents, for bikes and for skis, as well as images for both. 

Now let's check the contents of the directory with our documents Note that this is an internal staging area but it could also be an external S3 location, so there is no need to actually copy the PDFs into Snowflake. 

```SQL
SELECT * FROM DIRECTORY('@DOCS');
```
### PDFs

To parse the documents, we are going to use the native Cortex [PARSE_DOCUMENT](https://docs.snowflake.com/en/sql-reference/functions/parse_document-snowflake-cortex). (For LAYOUT we can specify OCR or LAYOUT.) In our case, we will be using LAYOUT so the content is in markdown formatted.

```SQL
CREATE OR REPLACE TEMPORARY TABLE RAW_TEXT AS
SELECT 
    RELATIVE_PATH,
    TO_VARCHAR (
        SNOWFLAKE.CORTEX.PARSE_DOCUMENT (
            '@DOCS',
            RELATIVE_PATH,
            {'mode': 'LAYOUT'} ):content
        ) AS EXTRACTED_LAYOUT 
FROM 
    DIRECTORY('@DOCS')
WHERE
    RELATIVE_PATH LIKE '%.pdf';

SELECT * FROM RAW_TEXT;
```

Next we are going to split the contents of the PDF files into chunks with some overlap to make sure information and context is not lost. You can read more about token limits and text splitting in the [Cortex Search Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview).

Create the table that will be used by Cortex Search service as a tool for Cortex Agents in order to retrieve information from PDF and JPEG files.

```SQL
create or replace TABLE DOCS_CHUNKS_TABLE ( 
    RELATIVE_PATH VARCHAR(16777216), -- Relative path to the PDF file
    CHUNK VARCHAR(16777216), -- Piece of text
    CHUNK_INDEX INTEGER, -- Index for the text
    CATEGORY VARCHAR(16777216) -- Will hold the document category to enable filtering
);
```

To split the text we also use the native Cortex [SPLIT_TEXT_RECURSIVE_CHARACTER](https://docs.snowflake.com/en/sql-reference/functions/split_text_recursive_character-snowflake-cortex). 

```SQL
-- Create chunks from extracted content
insert into DOCS_CHUNKS_TABLE (relative_path, chunk, chunk_index)

    select relative_path, 
            c.value::TEXT as chunk,
            c.INDEX::INTEGER as chunk_index
            
    from 
        raw_text,
        LATERAL FLATTEN( input => SNOWFLAKE.CORTEX.SPLIT_TEXT_RECURSIVE_CHARACTER (
              EXTRACTED_LAYOUT,
              'markdown',
              1512,
              256,
              ['\n\n', '\n', ' ', '']
           )) c;

SELECT * FROM DOCS_CHUNKS_TABLE;
```

Now let's see how CLASSIFY_TEXT Cortex function can be used to classify the document type. We have two classes Bike and Snow, and we will pass the document title and the first chunk of the document to the function.

```SQL
CREATE OR REPLACE TEMPORARY TABLE docs_categories AS WITH unique_documents AS (
  SELECT
    DISTINCT relative_path, chunk
  FROM
    docs_chunks_table
  WHERE 
    chunk_index = 0
  ),
 docs_category_cte AS (
  SELECT
    relative_path,
    TRIM(snowflake.cortex.CLASSIFY_TEXT (
      'Title:' || relative_path || 'Content:' || chunk, ['Bike', 'Snow']
     )['label'], '"') AS category
  FROM
    unique_documents
)
SELECT
  *
FROM
  docs_category_cte;
```

Let's review the categories:

```SQL
select * from docs_categories;
```

And now let's update the table:

```SQL
update docs_chunks_table 
  SET category = docs_categories.category
  from docs_categories
  where  docs_chunks_table.relative_path = docs_categories.relative_path;
```

### Images

Now let's process the images we have for our bikes and skis. We are going to use the COMPLETE multi-modal function asking for an image description and classification. We will add this information into the DOCS_CHUNKS_TABLE where we also have the PDF documentation.

```SQL
insert into DOCS_CHUNKS_TABLE (relative_path, chunk, chunk_index, category)
SELECT 
    RELATIVE_PATH,
    CONCAT('This is a picture describing the bike: '|| RELATIVE_PATH || 
        'THIS IS THE DESCRIPTION: ' ||
        SNOWFLAKE.CORTEX.COMPLETE('claude-3-5-sonnet',
        'DESCRIBE THIS IMAGE: ',
        TO_FILE('@DOCS', RELATIVE_PATH))) as chunk,
    0,
    SNOWFLAKE.CORTEX.COMPLETE('claude-3-5-sonnet',
        'Classify this image, respond only with Bike or Snow: ',
        TO_FILE('@DOCS', RELATIVE_PATH)) as category,
FROM 
    DIRECTORY('@DOCS')
WHERE
    RELATIVE_PATH LIKE '%.jpeg';
```

Check the images descriptions created. These will be used the tool to retrieve information when needed:

```SQL
select * from DOCS_CHUNKS_TABLE
    where RELATIVE_PATH LIKE '%.jpeg';
```

### Enable Cortex Search Service

Now that we have processed the PDF and images, we can create a [Cortex Search Service](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview) that will automatically create embeddings and indexes over the chunks of text extracted. (Read the docs for the different embedding models available.)

When creating the service, we also specify what is the TARGET_LAG. This is the frequency used by the service to maintain the service with new or deleted data. Refer to the [Automatic Processing of New Documents](https://quickstarts.snowflake.com/guide/ask_questions_to_your_own_documents_with_snowflake_cortex_search/index.html?index=..%2F..index#6) section to understand how easy is to maintain your RAG when using Cortex Search.

```SQL
create or replace CORTEX SEARCH SERVICE DOCUMENTATION_TOOL
ON chunk
ATTRIBUTES relative_path, category
warehouse = COMPUTE_WH
TARGET_LAG = '1 hour'
EMBEDDING_MODEL = 'snowflake-arctic-embed-l-v2.0'
as (
    select chunk,
        chunk_index,
        relative_path,
        category
    from docs_chunks_table
);
```

## Step 3: Setup Structured Data Tool

Another tool that we will setup is Cortex Analyst. It will provide the capability to extract information from structured data stored in Snowflake tables. In the API call we will provider the location of our semantic file that contains information about the business terminology used to describe the data.

First, let's create some tables and generate data that provides additional context about bikes and ski products.

**DIM_ARTICLE – Article/Item Dimension**

Purpose: Stores descriptive information about the products (articles) being sold.

Key Columns:

    ARTICLE_ID (Primary Key): Unique identifier for each article.
    ARTICLE_NAME: Full name/description of the product.
    ARTICLE_CATEGORY: Product category (e.g., Bike, Skis, Ski Boots).
    ARTICLE_BRAND: Manufacturer or brand (e.g., Mondracer, Carver).
    ARTICLE_COLOR: Dominant color for the article.
    ARTICLE_PRICE: Standard unit price of the article.
    DIM_CUSTOMER – Customer Dimension

**DIM_CUSTOMER – Customer Dimension**

Purpose: Contains demographic and segmentation info about each customer.

Key Columns:

    CUSTOMER_ID (Primary Key): Unique identifier for each customer.
    CUSTOMER_NAME: Display name for the customer.
    CUSTOMER_REGION: Geographic region (e.g., North, South).
    CUSTOMER_AGE: Age of the customer.
    CUSTOMER_GENDER: Gender (Male/Female).
    CUSTOMER_SEGMENT: Marketing segment (e.g., Premium, Regular, Occasional).
    FACT_SALES – Sales Transactions Fact Table

**FACT_SALES – Sales Transactions Fact Table**

Purpose: Captures individual sales transactions (facts) with references to article and customer details.

Key Columns:

    SALE_ID (Primary Key): Unique identifier for the transaction.
    ARTICLE_ID (Foreign Key): Links to DIM_ARTICLE.
    CUSTOMER_ID (Foreign Key): Links to DIM_CUSTOMER.
    DATE_SALES: Date when the sale occurred.
    QUANTITY_SOLD: Number of units sold in the transaction.
    TOTAL_PRICE: Total transaction value (unit price × quantity).
    SALES_CHANNEL: Sales channel used (e.g., Online, In-Store, Partner).
    PROMOTION_APPLIED: Boolean indicating if the sale involved a promotion or discount.

Let's create these tables and insert sample data:

**DIM_ARTICLE**

```SQL
CREATE OR REPLACE TABLE DIM_ARTICLE (
    ARTICLE_ID INT PRIMARY KEY,
    ARTICLE_NAME STRING,
    ARTICLE_CATEGORY STRING,
    ARTICLE_BRAND STRING,
    ARTICLE_COLOR STRING,
    ARTICLE_PRICE FLOAT
);

INSERT INTO DIM_ARTICLE (ARTICLE_ID, ARTICLE_NAME, ARTICLE_CATEGORY, ARTICLE_BRAND, ARTICLE_COLOR, ARTICLE_PRICE)
VALUES 
(1, 'Mondracer Infant Bike', 'Bike', 'Mondracer', 'Green', 3000),
(2, 'Premium Bicycle', 'Bike', 'Veloci', 'Red', 9000),
(3, 'Ski Boots TDBootz Special', 'Ski Boots', 'TDBootz', 'White', 600),
(4, 'The Ultimate Downhill Bike', 'Bike', 'Graviton', 'Red', 10000),
(5, 'The Xtreme Road Bike 105 SL', 'Bike', 'Xtreme', 'Grey', 8500),
(6, 'Carver Skis', 'Skis', 'Carver', 'White', 790),
(7, 'Outpiste Skis', 'Skis', 'Outpiste', 'Grey', 900),
(8, 'Racing Fast Skis', 'Skis', 'RacerX', 'Grey', 950);
```

**DIM_CUSTOMER**

```SQL
CREATE OR REPLACE TABLE DIM_CUSTOMER (
    CUSTOMER_ID INT PRIMARY KEY,
    CUSTOMER_NAME STRING,
    CUSTOMER_REGION STRING,
    CUSTOMER_AGE INT,
    CUSTOMER_GENDER STRING,
    CUSTOMER_SEGMENT STRING
);

INSERT INTO DIM_CUSTOMER (CUSTOMER_ID, CUSTOMER_NAME, CUSTOMER_REGION, CUSTOMER_AGE, CUSTOMER_GENDER, CUSTOMER_SEGMENT)
SELECT 
    SEQ4() AS CUSTOMER_ID,
    'Customer ' || SEQ4() AS CUSTOMER_NAME,
    CASE MOD(SEQ4(), 5)
        WHEN 0 THEN 'North'
        WHEN 1 THEN 'South'
        WHEN 2 THEN 'East'
        WHEN 3 THEN 'West'
        ELSE 'Central'
    END AS CUSTOMER_REGION,
    UNIFORM(18, 65, RANDOM()) AS CUSTOMER_AGE,
    CASE MOD(SEQ4(), 2)
        WHEN 0 THEN 'Male'
        ELSE 'Female'
    END AS CUSTOMER_GENDER,
    CASE MOD(SEQ4(), 3)
        WHEN 0 THEN 'Premium'
        WHEN 1 THEN 'Regular'
        ELSE 'Occasional'
    END AS CUSTOMER_SEGMENT
FROM TABLE(GENERATOR(ROWCOUNT => 5000));
```

**FACT_SALES**

```SQL
CREATE OR REPLACE TABLE FACT_SALES (
    SALE_ID INT PRIMARY KEY,
    ARTICLE_ID INT,
    DATE_SALES DATE,
    CUSTOMER_ID INT,
    QUANTITY_SOLD INT,
    TOTAL_PRICE FLOAT,
    SALES_CHANNEL STRING,
    PROMOTION_APPLIED BOOLEAN,
    FOREIGN KEY (ARTICLE_ID) REFERENCES DIM_ARTICLE(ARTICLE_ID),
    FOREIGN KEY (CUSTOMER_ID) REFERENCES DIM_CUSTOMER(CUSTOMER_ID)
);

-- Populating Sales Fact Table with new attributes
INSERT INTO FACT_SALES (SALE_ID, ARTICLE_ID, DATE_SALES, CUSTOMER_ID, QUANTITY_SOLD, TOTAL_PRICE, SALES_CHANNEL, PROMOTION_APPLIED)
SELECT 
    SEQ4() AS SALE_ID,
    A.ARTICLE_ID,
    DATEADD(DAY, UNIFORM(-1095, 0, RANDOM()), CURRENT_DATE) AS DATE_SALES,
    UNIFORM(1, 5000, RANDOM()) AS CUSTOMER_ID,
    UNIFORM(1, 10, RANDOM()) AS QUANTITY_SOLD,
    UNIFORM(1, 10, RANDOM()) * A.ARTICLE_PRICE AS TOTAL_PRICE,
    CASE MOD(SEQ4(), 3)
        WHEN 0 THEN 'Online'
        WHEN 1 THEN 'In-Store'
        ELSE 'Partner'
    END AS SALES_CHANNEL,
    CASE MOD(SEQ4(), 4)
        WHEN 0 THEN TRUE
        ELSE FALSE
    END AS PROMOTION_APPLIED
FROM DIM_ARTICLE A
JOIN TABLE(GENERATOR(ROWCOUNT => 10000)) ON TRUE
ORDER BY DATE_SALES;
```

In the next section, we're going to explore the Semantic model file. We have provided two files that you can copy from the GIT repository:

```SQL
create or replace stage semantic_files ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = ( ENABLE = true );

COPY FILES
    INTO @semantic_files/
    FROM @DASH_CORTEX_AGENTS_SUMMIT.PUBLIC.git_repo/branches/main/
    FILES = ('semantic.yaml', 'semantic_search.yaml');
```

## Step 4: Explore the Semantic Model

The [semantic model](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/semantic-model-spec) maps business terminology to the structured data and adds contextual meaning. It allows [Cortex Analyst](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst) to generate the correct SQL for a question asked in natural language.

### Open the semantic model

Under **AI & ML** -> **Studio**, select **Cortex Analyst**

![image](img/3_cortex_analyst.png)

Select the existing semantic.yaml file

![image](img/4_select_semantic_file.png)

### Test the semantic model

You can try some analytical questions to test your semantic file:

- **What is the average revenue per transaction per sales channel?**
- **What products are often bought by the same customers?**

### Cortex Analyst and Cortex Search Integration

Improve Tool Usage with Dynamic Literal Retrieval

Using Cortex Analyst integration with Cortex Search, we can improve the retrieval of possible values of a column without listing them all in the semantic model file. Let's try it as example for the ARTICLE NAMES.

Click on DIM_ARTICLE -> Dimensions and edit ARTICLE_NAME. In the Dimension you will see that some sample values have been provided.

![image](img/5_sample_values.png)

Let's see what happens if we ask the following question:

- **What are the total sales for the carvers?**

You may see this response:

![image](img/6_response.png)

Now let's see what happens when we integrate the ARTICLE_NAME dimension with the Cortex Search Service we created in the Notebook (_ARTICLE_NAME_SEARCH). If you haven't run it already in the Notebook, execute this:

```SQL
CREATE OR REPLACE CORTEX SEARCH SERVICE _ARTICLE_NAME_SEARCH
  ON ARTICLE_NAME
  WAREHOUSE = COMPUTE_WH
  TARGET_LAG = '1 hour'
  EMBEDDING_MODEL = 'snowflake-arctic-embed-l-v2.0'
AS (
  SELECT
      DISTINCT ARTICLE_NAME
  FROM DIM_ARTICLE
);
```

Now back in the Semantic model UI:

- Remove the sample values provided
- Click on + Search Service and add _ARTICLE_NAME_SEARCH

It should look like this:

![image](img/7_cortex_search_integration.png)

Click on Save, also save your semantic file (top right) and ask the same question again:

- **What are the total sales for the carvers?**

Notice that now Cortex Analyst is able to provide the right answer because of the Cortex Search integration, we asked for "Carvers" but found that the correct article to ask about is "Carver Skis":

![image](img/8_right_answer.png)

## Step 5: Streamlit Application

Now that we have the tools ready, we can create a Streamlit app that puts it all together using [Cortex Agents API](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents) API.

We are going to leverate code from: [streamlit_app.py](https://github.com/Snowflake-Labs/sfguide-build-data-agents-using-snowflake-cortex-ai/blob/main/streamlit_app.py)

Click on **Projects** -> **Streamlit** -> **Streamlit App** and select "Create from repository" as shown below.

![image](img/9_create_app.png)

![image](img/9_create_app_2.png)

Select the **streamlit_app.py** file and click on **Create**.

![image](img/9_create_app_3.png)

![image](img/9_create_app_4.png)

## Step 6: Run Application

Open the Streamlit app and let's check it out.

### Unstructured data questions

These are questions where the answers can be found in the PDF documents.

- **What is the guarantee of the premium bike?**

![image](img/10_unstructured_question.png)

The code contains a *display_citations()* function as an example to show what pieces of information the Cortex Agent used to answer the question. In this case, we can see how it cites the warranty information extracted from the PDF file. 

Let's try these other questions.

- **What is the length of the carver skis?**

![image](img/10_carvers_question.png)

Since we have processed images, the extracted descriptions can also be used by Cortex Agents to answer questions. Here's one example:

- **Is there any brand in the frame of the downhill bike?**

![image](img/10_bikes_question.png)

Fell free to explore the PDF documents and image files to ask your own questions.

### Structured data questions

These are analytical questions where the answers can be found in structured data stored in Snowflake tables.

- **How much are we selling for the carvers per year for the North region?**

Notice that for this query, all 3 tables are used. Also note that the Cortex Search integration in the semantic model understands that the article name is "Carver Skis".

![image](img/11_carver_query.png)

Let's try these other questions.

- **How many infant bikes are we selling per month?**
- **What are the top 5 customers buying the carvers?**

Now observe the behavior of the following question:

- **What are the monthly sales via online for the racing fast in central?**

Cortex Agents API sends the request to Cortex Analyst, but it is not able to filter by `customer_region`:

![image](img/14_central_question.png)

If we take a look at the [semantic file](https://github.com/Snowflake-Labs/sfguide-build-data-agents-using-snowflake-cortex-ai/blob/main/semantic_search.yaml) we can see that `Central` is not included as one of the sample values:

```code
      - name: CUSTOMER_REGION
        expr: CUSTOMER_REGION
        data_type: VARCHAR(16777216)
        sample_values:
          - North
          - South
          - East
        description: Geographic region where the customer is located.
```

This would be a good opportunity to fine tune the semantic model. Either adding all possible values if there aren't many, or use Cortex Search as we have done before for the ARTICLE column.

## Step 7: Cortex Agents API

When calling the [Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents) API, we define the tools the Agent can use in that call. You can read the simple [Streamlit App](https://github.com/Snowflake-Labs/sfguide-build-data-agents-using-snowflake-cortex-ai/blob/main/streamlit_app.py) you set up to understand the basics before trying to create something more elaborat and complex.

We define the **API_ENDPOINT** for the agent, and how to access the different tools its going to use. In this lab, we have two Cortex Search services to retrieve information from PDFs about bikes and skis, and one Cortex Analyst service to retrieve analytical information from Snowflake tables. The Cortex Search services were created in the Notebook and the Cortex Analyst uses the semantic model we verified earlier.

![image](img/12_api_1.png)

All of these services are added to the payload sent to the Cortex Agents API. We also provide the model we want to use to build the final response, the tools to be used, and any specific instructions for generating the response.

## Step 8: (Optional) Integrations

Learn how to integrate Cortex Agents in [Slack](https://quickstarts.snowflake.com/guide/integrate_snowflake_cortex_agents_with_slack/index.html), [Microsoft Teams](https://quickstarts.snowflake.com/guide/integrate_snowflake_cortex_agents_with_microsoft_teams/index.html), and [Zoom](https://quickstarts.snowflake.com/guide/integrate-snowflake-cortex-agents-with-zoom/index.html).

## Step 9: Conclusion And Resources

Congratulations! You've learned how to securely build data agents and agentic applications in Snowflake.

### What You Learned

- How to setup your environment using Git integration and Snowflake Notebooks 
- How to work with semantic models and setup Cortex Analyst for structured data
- How to setup Cortext Search for unstructured data like PDFs and images
- How to use Cortex Agents REST API that uses these tools in a Streamlit application

### Related Resources

- [GitHub repo](https://github.com/Snowflake-Labs/sfguide-build-data-agents-using-snowflake-cortex-ai)
- [Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [Cortex Analyst](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- [Cortex Search](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview)
