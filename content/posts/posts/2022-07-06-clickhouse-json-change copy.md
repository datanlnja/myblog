---
title: "google colab in online education"
date: 2021-01-06T00:00:00+08:00
---

In my courses and at work I often use Google Colab, with this tool you can write not only Pure Python code, but also deploy full-fledged applications, for debugging or development purposes of course. For this purpose I use a well-known trick, when a local host is tipped to the internet, for example via ngrok (free service at the time of writing).

Google Colab itself is a very small virtual machine with limited permissions and resource consumption, however it can understand simple bash commands and networking, which is very useful for us.

Let me show you how it can be realised on the example of Superset service.

Go to the official site with the installation documentation, and enter the command set into our Google Colab. The use of ! should be used as a sign that we want to execute the bash script, nohup will start the process as a daemon.

```bash
!pip install apache-superset
!superset db upgrade
!superset fab create-admin (you have to enter the user and password manually)
!superset load_examples
!superset init
!nohup superset run -p 8088 --with-threads --reload --debugger
```

[Link](https://colab.research.google.com/drive/1CsyqZ7bPwyfoMkhOjv0ALxk8_KOh5E2j?usp=sharing) to this laptop so you can run and test it yourself.

Then install ngrok to translate ports from the local host to the internet

```bash
!pip install pyngrok
# token https://dashboard.ngrok.com/get-started/setup
!ngrok authtoken <YOUR TOKEN> 

# This command just maps the web face to another address
# You can find it at https://dashboard.ngrok.com/endpoints/status
# Each time you switch off the link will change
!nohup ngrok http -log=stdout 8088 > /dev/null &
```

Result: it opens on the internet and can be shown to anyone.

*Other examples of laptops*

- [Airflow](https://colab.research.google.com/drive/1Im7wHJx10n1VgObbX_0qKHNpRJCsVLPW?usp=sharing)
- [ClickHouse](https://colab.research.google.com/drive/1xOJKraWREPtDZZknzhfgKWyBxn9vH2dm?usp=sharing)
