# Data processing pipelines for Small Big Data

Small Big Data is a grey area in data science between "it fits in memory" and
100 Tb. Some of the tools used for big data are overkill, and they might require
a particular set of expertise that not every organization has. In contrast, many
of the libraries and paradigms used for small data can become expensive when
deploying to the cloud. How can we process large-ish data fast and efficiently? 

In this talk, we will cover some of the techniques, tools, and libraries that
have helped us process, extract relevant features, and serve large-ish data
without incurring some significant expenses. 

We will start reviewing some of the more fundamental built-in tools that Python
offers, like iterators/generators, threads, or async/await functions. 

We will continue going over how to leverage some third-party libraries, like
SQLAlchemy, Numba, or Dask, to run more memory and time-efficient data ingestion
pipelines. We will discuss custom merging/deduplication of incoming data, the
use of JIT compilers on certain parts of the process, and database bulk
operations to illustrate our usage of these libraries and their benefits. 

Feature stores are a hot topic nowadays, but most solutions require more
specialized services like Apache Hive or Cassandra. We will describe our
solution during this part of the talk with concrete usage examples. 

Finally, we will go through our deployment strategy. We will explain how we
leverage Airflow, Docker, and Kubernetes to run our data pipelines in
production, and how we utilize GitLab CI/CD pipelines to deploy them to the
cloud (in our case AWS.


---

**NOTE**
Since I recorded the presentation, my colleagues told that maybe Jupyter
notebooks might be a better way to show case the examples instead of plain
python scripts. When you see on the presentation `python app_*.py`, the content
of this script has been moved into one of the two notebooks here.

---


## Extra Resources

- <https://switowski.com/blog/ask-for-permission-or-look-before-you-leap>
- <https://rushter.com/blog/numba-cython-python-optimization/>
- <https://realpython.com/products/cpython-internals-book/>
- <https://realpython.com/intro-to-python-threading/>
- <https://leimao.github.io/blog/Python-Concurrency-High-Level/>
- <http://masnun.rocks/2016/10/06/async-python-the-different-forms-of-concurrency/>
- <https://docs.sqlalchemy.org/en/13/faq/performance.html#i-m-inserting-400-000-rows-with-the-orm-and-it-s-really-slow>
