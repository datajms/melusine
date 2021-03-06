# Melusine

<!-- <img src=`docs/_static/melusine.png` width=`200`/> -->
![](docs/_static/melusine.png)

[![pypi badge](https://img.shields.io/pypi/v/melusine.svg)](https://pypi.python.org/pypi/melusine)
[![](https://img.shields.io/travis/sachasamama/melusine.svg)](https://travis-ci.org/sachasamama/melusine)
[![documentation badge](https://readthedocs.org/projects/melusine/badge/?version=latest)](https://readthedocs.org/projects/melusine/)

- Free software: Apache Software License 2.0
- Documentation: [https://melusine.readthedocs.io](https://melusine.readthedocs.io).

# Overview

**Melusine** is a high-level Scikit-Learn API for emails classification and feature extraction,
written in Python and capable of running on top of Scikit-Learn, Keras or Tensorflow.
It was developed with a focus on emails written in french.

Use **Melusine** if you need a library which :
  * Supports both convolutional networks and recurrent networks, as well as combinations of the two.
  * Runs seamlessly on CPU and GPU.

**Melusine** is compatible with `Python >= 3.5`.


## The Melusine package

This package is designed for the preprocessing, classification and automatic summarization of emails written in french.

<!-- <img src=`docs/_static/schema_1.png` width=`600`/> -->
![](docs/_static/schema_1.png =700x)

**3 main subpackages are offered :**

* ``prepare_email`` : to preprocess and clean the emails.
* ``summarizer`` : to extract keywords from an email.
* ``models`` : to classify e-mails according to categories pre-defined defined by the user.

**2 other subpackages are offered as building blocks :**

* ``nlp_tools`` : to provide classic NLP tools such as tokenizer, phraser and embeddings.
* ``utils`` : to provide a *TransformerScheduler* class to build your own transformer and integrate it into a scikit-learn Pipeline.

**An other subpackage is also provided** to manage, modify or add parameters such as : regular expressions, keywords, stopwords, etc.

* ``config`` : contains *`ConfigJsonReader`* class to setup and handle a *conf.json* file. This JSON file is the core of this package since it's used by different submodules to preprocess the data.


## Getting started: 30 seconds to Melusine

### Installation

```
pip install melusine
```

To use Melusine in a project

```python
import melusine
```

### Input data : Email DataFrame

The basic requirement to use Melusine is to have an input e-mail DataFrame with the following columns:

- *body*   : Body of an email (single message or conversation historic)
- *header* : Header/Subject of an email
- *date*   : Reception date of an email
- *from*   : Email address of the sender
- *to*     : Email address of the recipient
- *label* (optional): Label of the email for a classification task (examples: Business, Spam, Finance or Family)

| body                       | header         | date                           | from                         | to                             | label   |
|:---------------------------|:--------------:|:------------------------------:|:----------------------------:|:-------------------------------------:|:-------:|
| Thank you.\\nBye,\\nJohn   | Re: Your order | jeudi 24 mai 2018 11 h 49 CEST | anonymous.sender@unknown.com | anonymous.recipient@unknown.fr | label_1 |

### Pre-processing pipeline

A working pre-processing pipeline is given below::

```python
from sklearn.pipeline import Pipeline
from melusine.utils.transformer_scheduler import TransformerScheduler
from melusine.prepare_email.manage_transfer_reply import check_mail_begin_by_transfer
from melusine.prepare_email.manage_transfer_reply import update_info_for_transfer_mail
from melusine.prepare_email.manage_transfer_reply import add_boolean_answer
from melusine.prepare_email.manage_transfer_reply import add_boolean_transfer
from melusine.prepare_email.build_historic import build_historic
from melusine.prepare_email.mail_segmenting import structure_email
from melusine.prepare_email.body_header_extraction import extract_last_body
from melusine.prepare_email.cleaning import clean_body
from melusine.prepare_email.cleaning import clean_header

ManageTransferReply = TransformerScheduler(
functions_scheduler=[
    (check_mail_begin_by_transfer, None, ['is_begin_by_transfer']),
    (update_info_for_transfer_mail, None, None),
    (add_boolean_answer, None, ['is_answer']),
    (add_boolean_transfer, None, ['is_transfer'])
])

EmailSegmenting = TransformerScheduler(
functions_scheduler=[
    (build_historic, None, ['structured_historic']),
    (structure_email, None, ['structured_body'])
])

Cleaning = TransformerScheduler(
functions_scheduler=[
    (extract_last_body, None, ['last_body']),
    (clean_body, None, ['clean_body']),
    (clean_header, None, ['clean_header'])
])

prepare_data_pipeline = Pipeline([
  ('ManageTransferReply', ManageTransferReply),
  ('EmailSegmenting', EmailSegmenting),
  ('Cleaning', Cleaning),
])

df_email = prepare_data_pipeline.fit_transform(df_email)
```

In this example, the pre-processing functions applied are:

- ``check_mail_begin_by_transfer`` : Email is a direct transfer (True/False)
- ``update_info_for_transfer_mail`` : Update body, header, from, to, date if direct transfer
- ``add_boolean_answer`` : Email is an answer (True/False)
- ``add_boolean_transfer`` : Email is transferred (True/False)
- ``build_historic`` : When email is a conversation, reconstructs the individual message historic
- ``structure_email`` : Splits parts of each messages in historic and tags them (tags: Hello, Body, Greetings, etc)

### Phraser and Tokenizer pipeline

A pipeline to train and apply the phraser end tokenizer is given below::

```python
from melusine.nlp_tools.phraser import Phraser
from melusine.nlp_tools.tokenizer import Tokenizer

phraser = Phraser(columns='clean_body')
phraser.train(df_email)
phraser.save('./phraser.pkl')
phraser = Phraser().load('./phraser.pkl')

PhraserTransformer = TransformerScheduler(
functions_scheduler=[
    (phraser_on_body, (phraser,), ['clean_body']),
    (phraser_on_header, (phraser,), ['clean_header'])
])

phraser_tokenizer_pipeline = Pipeline([
  ('PhraserTransformer', PhraserTransformer),
  ('Tokenizer', Tokenizer(columns=['clean_body', 'clean_header']))
])

df_email = phraser_tokenizer_pipeline.fit_transform(df_email)
```

### Embeddings training

An example of embedding training is given below::

```python
from melusine.nlp_tools.embedding import Embedding

embedding = Embedding(columns='clean_body')
embedding.train(df_email)
embedding.save('./embedding.pkl')
```

### Metadata pipeline

A pipeline to prepare the metadata is given below:

```python
from melusine.prepare_email.metadata_engineering import MetaExtension
from melusine.prepare_email.metadata_engineering import MetaDate
from melusine.prepare_email.metadata_engineering import Dummifier

metadata_pipeline = Pipeline([
  ('MetaExtension', MetaExtension()),
  ('MetaDate', MetaDate()),
  ('Dummifier', Dummifier(columns_to_dummify=['extension', 'dayofweek', 'hour']))
])

df_meta = metadata_pipeline.fit_transform(df_email)
```

### Keywords extraction

An example of keywords extraction is given below::

```python
from melusine.summarizer.keywords_generator import KeywordsGenerator

keywords_generator = KeywordsGenerator()
df_email = phraser_tokenizer_pipeline.fit_transform(df_email)
```

### Classification

An example of classification is given below::
```python
from sklearn.preprocessing import LabelEncoder
from melusine.nlp_tools.embedding import Embedding
from melusine.models.neural_architectures import cnn_model
from melusine.models.train import NeuralModel

X = df_email.drop(['label'], axis=1)
y = df_email.label

le = LabelEncoder()
y = le.fit_transform(y)

pretrained_embedding = Embedding().load(./embedding.pkl)

nn_model = NeuralModel(neural_architecture_function=cnn_model,
                       pretrained_embedding=pretrained_embedding)
nn_model.fit(X, y)
y_res = nn_model.transform(X_test)
```

## Glossary

### Pandas dataframes columns

Because Melusine manipulates pandas dataframes, the naming of the columns is imposed.
Here is a basic glossary to provide an understanding of each columns manipulated.
Initial columns of the dataframe:

* **body :** the body of the email. It can be composed of a unique message, a historic of messages, a transfer of messages or a combination of historics and transfers.
* **header :** the subject of the email.
* **date :** the date the email has been sent. It corresponds to the date of the last message of the email has been written.
* **from :** the email address of the author of the last message of the email.
* **to :** the email address of the recipient of the last message.

Columns added by Melusine:

* **is_begin_by_transfer :** boolean, indicates if the email is a direct transfer. In that case it is recommended to update the value of the initial columns with the informations of the message transferred.
* **is_answer :** boolean, indicates if the email contains a historic of messages
* **is_transfer :** boolean, indicates if the email is a transfer (in that case it does not have to be a direct transfer).
* **structured_historic :** list of dictionaries, each dictionary corresponds to a message of the email. The first dictionary corresponds to the last message (the one that has been written) while the last dictionary corresponds to the first message of the historic. Each dictionary has two keys :

  - *meta :* to access the metadata of the message as a string.
  - *text :* to access the message itself as a string.

* **structured_body :** list of dictionaries, each dictionary corresponds to a message of the email. The first dictionary corresponds to the last message (the one that has been written) while the last dictionary corresponds to the first message of the historic. Each dictionary has two keys :

  - *meta :* to access the metadata of the message as a dictionary. The dictionary has three keys:
    + *date :* the date of the message.
    + *from :* the email address of the author of the message.
    + *to :* the email address of the recipient of the message.

  - *text :* to access the message itself as a dictionary. The dictionary has two keys:
    + *header :* the subject of the message.
    + *structured_text :* the different parts of the message segmented and tagged as a list of dictionaries. Each dictionary has two keys:
      - *part :* to access the part of the message as a string.
      - *tags :* to access the tag of the part of the message.


* **last_body :** string, corresponds to the part of the last message of the email that has been tagged as `BODY`.
* **clean_body :** string, corresponds a cleaned last_body.
* **clean_header :** string, corresponds to a cleaned header.
* **clean_text :** string, concatenation of clean_header and clean_body.
* **tokens :** list of strings, corresponds to a tokenized column, by default clean_text.
* **keywords :** list of strings, corresponds to the keywords of extracted from the tokens column.

### Tags

Each messages of an email are segmented the in the **structured_body** columns and each parts are assigned a tag:

* `RE/TR` : any metadata such as date, from, to etc.
* `DISCLAIMER` : any disclaimer such as `L'émetteur décline toute responsabilité...`.
* `GREETINGS` : any greetings such as `Salutations`.
* `PJ` : any indication of an attached document such as `See attached file...`.
* `FOOTER` : any footer such as `Provenance : Courrier pour Windows`.
* `HELLO` : any salutations such as `Bonjour,`.
* `THANKS` : any thanks such as `Avec mes remerciements`
* `BODY` : the core of the the message which contains the valuable information.


## Motivation & history


### Origin of the project

**MAIF**, being one of the leading mutual insurance companies in France, receives daily a large volume of emails from its clients
and is under pressure to reply to their requests as efficiently as possible. As such an efficient routing system is of the
upmost importance to assign each emails to its right entity.
However the previously outdated routing system put the company under ever increasing difficulties to fulfill its pledge.
In order to face up to this challenge, MAIF in collaboration with **Quantmetry**, has implemented a new routing system
based on state-of-the-art NLP and Deep Learning techniques that would classify each email under the right label
according to its content and extract the relevant information to help the MAIF counsellors processing the emails.

### Ambitions of the project

**Melusine** is the first Open Source and free-of-use solution dedicated specifically to the qualification of e-mails written in french.
The ambition of this Python package is to become a reference, but also to live in the French NLP community by federating users and contributors.
Initially developed to answer the problem of routing e-mails received by the MAIF, the solution was implemented using state-of-the-art techniques in Deep Learning and NLP.
Melusine can be interfaced with Scikit-Learn: it offers the user the possibility to train his own classification and automatic summarization model according to the constraints of his problem.

### The collaboration between Quantmetry and MAIF

After collaborating for the implementation of its routing system with Quantmetry,
a pure player consulting firm in AI, MAIF pursued the partnership to develop the *Melusine* package.

### Why Melusine ?

Following MAIF's tradition to name its open source packages after deities, it was chosen to release this package
under the name of Melusine as an homage to a legend from the local folklore in the Poitou region in France
where MAIF is historically based.


## Credits

This package was created with [Cookiecutter](https://github.com/audreyr/cookiecutter) and the [audreyr/cookiecutter-pypackage](https://github.com/audreyr/cookiecutter-pypackage) project template.
