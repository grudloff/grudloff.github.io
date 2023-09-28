---
title: "Building a Personal Journal App with MongoDB and Streamlit"
categories:
  - Blog
---

In today's digital age, maintaining a personal journal has evolved from the traditional pen-and-paper approach to digital solutions that offer convenience and accessibility. Creating a custom personal journal app can be an exciting and rewarding endeavor, allowing you to combine modern technologies to capture your thoughts and experiences. In this blog post, we'll delve into the development journey of crafting a personal journal app using MongoDB and Streamlit.

## The Power of MongoDB and Streamlit
**MongoDB**: As a versatile NoSQL database, MongoDB offers a flexible and scalable solution for storing structured and semi-structured data. Its document-based model makes it ideal for applications like personal journaling, where each entry can have varying attributes.

**Streamlit**: Streamlit, on the other hand, empowers developers to rapidly create interactive and data-driven web applications using only Python. Its intuitive approach allows you to focus on functionality and presentation without the complexities of front-end development.

In this exploration of building a personal journal app, we'll uncover the process of setting up a MongoDB Atlas cluster for data storage and employing Streamlit to develop a user-friendly interface. Join us as we journey through the steps of connecting to the database, adding new entries, filtering entries by date, and providing the ability to modify or delete existing entries.

Whether you're an aspiring developer seeking to expand your skill set or someone passionate about journaling who wants to create a tailored digital experience, this blog post will guide you through the intricacies of merging MongoDB's database capabilities with Streamlit's web application development prowess.

Let's embark on this adventure of coding, creativity, and data storage as we bring to life a personalized digital journal using MongoDB and Streamlit.

## Setting Up MongoDB with Atlas

To kickstart the development of our personal journal app, we need a reliable and accessible database solution. MongoDB Atlas, the fully managed cloud database service offered by MongoDB, fits the bill perfectly. In this section, we'll walk through the process of setting up your MongoDB Atlas cluster and establishing a connection to it.

### Signing Up for MongoDB Atlas

If you're new to MongoDB Atlas, the first step is to create an account. Head over to the [MongoDB Atlas website](https://www.mongodb.com/atlas/database) and click on the "Get started free" button. Follow the registration process, providing the necessary information to create your account.

### Creating a New Cluster

Once you've successfully signed up and logged in to your MongoDB Atlas account, you'll be greeted with the dashboard. Here, you'll find an option to "Create a New Cluster." Click on this button to initiate the cluster creation process.

- **Cluster Configuration**: You'll be prompted to select your preferred cloud provider (Amazon, GCP or Azure), region, and other cluster-specific configurations. MongoDB Atlas offers a shared cluster tier, where the M0 instance with 512MB storage is free, which is suitable for small-scale projects like our personal journal app.

### Connecting Your App to the Cluster

With your cluster successfully created, the next step is to connect your personal journal app to the MongoDB Atlas cluster. Follow these steps:

1. **Access Database Access**: In the cluster dashboard, navigate to the "Database Access" tab. Here, you can create a new database user with the necessary privileges. This user will be used to authenticate your app's connection to the database. Copy the password, as you will not be able to see it later. If you didn't, there is no problem as you can set another password.

2. **Whitelist IP Address**: To enhance security, you can whitelist specific IP addresses that are allowed to access your MongoDB Atlas cluster. This helps prevent unauthorized access. In this simple case, security is not a big issue. Setting the IP to 0.0.0.0/0 will allow any IP to connect to the database.

3. **Get Connection String**: In the "Database" tab, click on the "Connect" button for your cluster. Then, select the "Drivers" options and select python as the driver. MongoDB Atlas will provide you with a connection string. Make sure to replace the placeholders in the connection string with the appropriate credentials you created earlier.

### Connecting via `pymongo`

In your Python app code, you can use the `pymongo` library to establish a connection to your MongoDB Atlas cluster. The connection string you obtained from MongoDB Atlas will be used as follows:

```python
from pymongo.mongo_client import MongoClient
uri = "mongodb+srv://<user>:<password>@<cluster>.<server>.mongodb.net/?retryWrites=true&w=majority"
# Create a new client and connect to the server
client = MongoClient(uri)
# Send a ping to confirm a successful connection
try:
    client.admin.command('ping')
    print("Pinged your deployment. You successfully connected to MongoDB!")
except Exception as e:
    print(e)
```

With this connection established, you're ready to interact with your MongoDB Atlas cluster using the `pymongo` library. Running this code is a good way to test that all the setting up was done correctly. 
Absolutely, let's start with the section on Streamlit, caching, and securing connection information. I'll provide explanations along with code snippets to illustrate the concepts.

---

### Create a Database and Collection

In the left-hand navigation menu, click on "Database" and then select the cluster you created.

Click the "Collections" tab.

In the "Collections" view, click the "Create Database" button. Provide a name for your database, such as "journal_entries." Also provide a name for your collection, such as "entries." Then, click on Create.

## MongoDB Connection: Streamlit, Caching, and Securing Connection Information

In the development of your personal journal app, establishing a secure and efficient connection to the MongoDB database is crucial. Let's dive into how you achieved this using Streamlit, caching, and securely stored connection information.

### Streamlit and Caching

Streamlit is an incredible tool for quickly building interactive web applications. It's particularly effective for prototypes and small-scale applications. One of its strengths is the caching system, which can greatly improve the app's performance by reusing expensive computations and database connections.

```python
# Initialize connection.
@st.cache_resource
def init_connection():
    uri = st.secrets["mongo"].uri
    return MongoClient(uri, server_api=ServerApi('1'))

client = init_connection()
```

In the code above, the `init_connection` function is decorated with `st.cache_resource`. This means that the function's return value, which is the MongoDB client instance in this case, will be cached and reused across different invocations of the app. This ensures that the database connection isn't established repeatedly, reducing overhead.

### Securing Connection Information

It's vital to keep sensitive information like database URIs secure. Streamlit provides a way to store and access such information securely using secrets. Here's how you can do it:

1. Store the MongoDB URI in the Streamlit secrets manager (`~/.streamlit/secrets.toml`):

```toml
[mongo]
uri = "your_actual_mongo_uri"
```

2. Access the secret within your app code:

```python
uri = st.secrets["mongo"].uri
```

By following this approach, you avoid hardcoding sensitive information directly in your code, reducing the risk of exposure.

---

## New Entry

In this section, we'll delve into the process of adding new journal entries to your Streamlit app. We'll cover how users can input their entries, how the app captures the input, and how the data is inserted into the MongoDB collection.

### Adding a New Entry

To allow users to add new journal entries, we provide a text area where they can type their thoughts. We've also included a button labeled "Add entry" that, when clicked, triggers the addition of the new entry. Let's take a look at the code responsible for this:

```python
st.subheader("New Entry")
entry = st.text_area(label="Entry", height=TEXT_HEIGHT, label_visibility="hidden")
submit = st.button("Add entry", use_container_width=True)

if submit and entry:
    date = datetime.now()
    client.journal_entries.entries.insert_one({"date": date, "entry": entry})
    get_data.clear()
    st.experimental_rerun()
```

In the above code:

- The `client.journal_entries.entries.insert_one` method inserts the new entry along with the date into the MongoDB collection.
-  After insertion, we clear the cache for `get_data` and use `st.experimental_rerun()` to refresh the UI and show the newly added entry.
> The cached function `get_data` is used to get previous entries and will be described in the following section.

## Modify Entries

In the Modify Entries section, we'll delve into how users can edit their existing journal entries. This functionality enhances the app's utility by allowing users to update their thoughts over time. To achieve this, we leverage Streamlit's session state to seamlessly switch between the view mode and the modification mode.

### Displaying Previous Entries

Before we dive into modifications, let's briefly revisit how the previous journal entries are displayed. We utilize MongoDB to fetch a set of entries based on user preferences, including the start and end dates and the number of entries to show.

```python
@st.cache_data(ttl=600)
def get_data(num_entries, start_date, end_date):
    start_date = datetime.combine(start_date, datetime.max.time())
    end_date = datetime.combine(end_date, datetime.min.time())
    try:
        items = client.journal_entries.entries.find(
            {"date": {"$gte": end_date, "$lte": start_date}},
            limit=num_entries,
        )
        items = list(items)

    except Exception as e:
        st.warning("Failed to load from MongoDB collection.")
        st.error(repr(e))
        items = []
    return items

# Sidebar to filter by date
st.sidebar.subheader("Filter by date")
# ... date input controls ...

# Number of entries slider
st.sidebar.subheader("Number of entries")
# ... entry number slider ...

# Fetch and display previous entries
st.subheader("Previous Entries")
items = get_data(num_entries, start_date, end_date)
for item in items:
    # ... display entry ...
    modify = st.button("Modify", key="modify_"+repr(item["_id"]),
                       use_container_width=True)
    if modify:
        # ... set session state variables ...
        st.experimental_rerun()
```

### Enabling Modification

When a user clicks the "Modify" button, we need to transition from the regular view mode to the modification mode. We achieve this by setting session state variables that indicate the modification process has started. This allows us to differentiate between the two modes and adjust the UI accordingly.

```python
if not st.session_state.get("modifying", False):
    # ... display new entry section ...
    for item in items:
        # ... display entry ...
        modify = st.button("Modify", key="modify_"+repr(item["_id"]),
                           use_container_width=True)
        if modify:
            st.session_state["modifying"] = True
            st.session_state["entry"] = item["entry"]
            st.session_state["date"] = item["date"]
            st.session_state["id"] = item["_id"]
            st.experimental_rerun()
else:
    # ... display modify entry section ...
```

### Modifying and Returning

Once in the modification mode, users can edit the content of the selected entry. Additionally, they have the option to either delete the entry or save the modifications. The "Return" button serves to exit the modification mode and return to the regular view.

```python
if not st.session_state.get("modifying", False):
    # ... regular view ...
else:
    # Modify an existing entry.
    header_columns = st.columns(3)

    header_columns[0].subheader("Modify Entry")
    header_columns[2].button("Return", key="come_back", use_container_width=True,
                             on_click = reset_modifying)
    id = st.session_state["id"]
    date = st.session_state["date"]
    entry = st.text_area("Entry", st.session_state["entry"], height=TEXT_HEIGHT,
                         label_visibility="hidden")

    col1, col2 = st.columns(2)

    delete = col1.button("Delete entry", use_container_width=True)
    if delete:
        # ... delete entry ...
    submit = col2.button("Modify entry", use_container_width=True)
    if submit:
        # ... modify entry ...
        st.session_state["modifying"] = False
        get_data.clear()
        st.experimental_rerun()
```

By carefully managing session state variables and using Streamlit's `st.experimental_rerun()` function, we've achieved a smooth transition between viewing and modifying journal entries. This user-friendly design ensures that users can seamlessly edit their entries without confusion. The next section will conclude our journey by summarizing the development process and highlighting key takeaways.

##  Deploy to Streamlit Community Cloud
1. Fork the [github repository](https://github.com/grudloff/Journal).
1. Visit the [Streamlit Community Cloud](https://streamlit.io/cloud).
2. Sign in with your GitHub account.
3. Click on the "New app" button.

Select your forked repository from the dropdown list, the name of the main (that in this case is 'journal.py'), and the URL you want for the webapp.

Then select "Advanced settings", and then input the 'uri' of your atlas cluster.

Click the "Deploy" button.

Streamlit Sharing will automatically pull the code from your GitHub repository, set up the environment based on the requirements.txt file, and run your app. Once the deployment process is complete, you'll receive a URL that you can share with others to access your personal journal app.


## Conclusion

The security of storing information on your own MongoDB cluster adds a crucial layer of protection to the sensitive and personal data captured in the journal app. With the data residing within your controlled environment, you mitigate potential risks associated with third-party data breaches and unauthorized access. This approach enables you to establish and enforce your security protocols, ensuring that your users' private thoughts and reflections remain confidential.

By hosting the data on your MongoDB cluster, you retain full control over access management, encryption, and authentication. You can implement encryption at rest and in transit, safeguarding the data against interception and unauthorized viewing. Furthermore, you have the autonomy to set up role-based access control, granting different levels of access to different users. This not only secures the information but also offers peace of mind to your app's users, knowing that their personal content is being treated with the utmost care.

Looking ahead, storing this journal data on your own cluster opens up a realm of exciting possibilities. The data can serve as a valuable resource for future processing, analysis, and insights. As natural language processing (NLP) and large language model (LLM) technologies continue to advance, your journal app's data can become a goldmine for sentiment analysis, trend identification, and even personalized recommendation systems.

By harnessing the power of LLM models, you could offer users insights into their emotional patterns over time, helping them identify trends in their own writing that might otherwise go unnoticed. Sentiment analysis could provide users with an understanding of their emotional states during different periods, fostering self-awareness and personal growth. Moreover, your app could potentially suggest related journal entries or prompts based on a user's writing style and content, enhancing the journaling experience even further.

Incorporating such advanced processing capabilities could turn your journal app into a dynamic tool for introspection and self-improvement. Users could receive actionable insights and personalized suggestions, making their journaling journey more insightful and rewarding.

In summary, the development of this personal journal app stands as a testament to the power of streamlined design and user-focused functionality. Through the thoughtful integration of MongoDB and the user-friendly Streamlit framework, creating a digital space for personal reflection has never been easier. The app's interface enables seamless journal entry creation, modification, and review, fostering a platform that promotes introspection and growth. For those seeking to embark on a similar journey, the beauty of this approach lies in its simplicity. The entire development process is encapsulated within this [GitHub repository](https://github.com/grudloff/Journal), making it readily accessible for anyone interested. By forking the repository, individuals can quickly adapt and personalize the app to align with their unique vision, allowing the benefits of digital journaling to be experienced by a wider audience.
