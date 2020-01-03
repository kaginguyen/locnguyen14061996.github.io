---
title: "Setting Up Airflow on Ubuntu 18.04 and Python 3.6.7- Step by Step Guide"
date: 2020-01-03
tags: [Data Science, Data Engineer]
---

Due to many reasons, I have finally able to convince my Manager and team to try out Airflow. Most team members except the Manager are already Python's users so it would not be too big of an issue to assemble all currently running jobs. 

With the amount of jobs running around is so high. It is surely better to get them inside a specific tool for ease in management. 

With that, let's get started in deploying. 

First, get yourself a Virtual Machine or server or whatever you can that can run continuously and reliably and Ubuntu 18.04 should be installed so we can begin. 

As beginning, lets start by upgrading your OS :
```bash
sudo apt install update
sudo apt install upgrade 
``` 

Lets check if you have python3 or not, cause support for python 2 is already dropped: 

```bash 
python3 -V
which python3 
``` 

This is assuming that Python 3 was installed and alias as python3. If you are using a different alias like python3.6 or python3.7 it would still work, just a little bit different, and don't worry, I will let you know if you have to change something with your python alias. 

Having a fresh Virtual Machine can be quite a bit incovenient like having no pip, so let's install it: 
```bash
sudo apt install python3-pip
``` 

Oke, seems like we are ready to start the job, let's go to ~HOME and start working: 
```bash
cd ~
pip3 install apache-airflow
``` 



