---
title: My first post
date: 2022-09-17 13:12 +0530
categories: [python,fileorganising,scripting]
tags: [python]     # TAG names should always be lowercase
---
# Hello this is a program to organise file based on type
#               the actual file: 
## All the imports
  ``python
from os import listdir
from os.path import isfile, join
import os
import shutil
  ``
##  Sorting Method
  ``python
  
def sort_files_in_a_folder(mypath):
    """
    A function to sort the files in a download folder
    into their respective categories
    """
    files = [f for f in listdir(mypath) if isfile(join(mypath, f))]
    file_type_variation_list = []
    filetype_folder_dict = {}
    for file in files:
        filetype = file.split(".")[1]
        if filetype not in file_type_variation_list:
            file_type_variation_list.append(filetype)
            computer = mypath + "/" + filetype + "_folder"
            filetype_folder_dict[str(filetype)] = str(computer)
            if os.path.isdir(computer) == True:  # folder exists
                continue
            else:
                os.mkdir(computer)
    for file in files:
        src_path = mypath + "/" + file
        filetype = file.split(".")[1]
        if filetype in filetype_folder_dict.keys():
            dest_path = filetype_folder_dict[str(filetype)]
            shutil.move(src_path, dest_path)
    print(src_path + ">>>" + dest_path)
  ``
## Main
``python
if __name__ == "__main__":
mypath = "/home/amogh/Downloads" #change this directory to the one you wish to organise the files in
sort_files_in_a_folder(mypath)
``
# Installation of the file 
``bash
 git clone https://github.com/amogh-dongre/python_projects 
``
# Config notes
 1. go into the python_projects folder. 
 2. open the organise.py file in a text editor of choice and change the directory path in the mypath variable.
 
