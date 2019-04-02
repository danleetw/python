# python
# -*- coding: utf-8 -*-
"""
Created on 

@author: Dan Lee
"""

#------------------Import Library ---------
import chardet
import os
import shutil

import requests
#conda install requests_ntlm
#!conda install -c conda-forge requests_ntlm 
#!conda install -c conda-forge xlrd
from requests_ntlm import HttpNtlmAuth


import configparser
import base64
import random


#------------------基本類 Function ---------
# ---傳回模組版本
def version():
    return('20190222')

# ---讀/寫參數檔案
def read_from_config(config_filename='config.ini',section='general',keyname='',ispassword=False,default_val='',write_back=False):
    configfile = configparser.ConfigParser()
    configfile.read(config_filename)
    value=''
    
    try:
        value=configfile.get(section,keyname)
        if ispassword==True:
            value=deCodePasswd(value)
    except Exception as e:
        print("Read Config:",config_filename," Section:",section, " Keyname:",keyname," Err Desc:",e)
        value=default_val
        
    if write_back==True:
        #print("Write-----",default_val)
        try:
           configfile.add_section(section)
        except Exception as e:
           pass 
       
        try:    
            #---寫入參數----
            writevalue=default_val
            if ispassword==True:
                writevalue=enCodePasswd(writevalue)
            
            configfile.set(section,keyname, writevalue)
            configfile.write(open(config_filename, "w"))
            #print("Write Config:",config_filename," Section:",section, " Keyname:",keyname," Value:",default_val)
        except Exception as e:
           print(e)
           pass 
        
    #print("KeyValue:",value)
    return(value)

#------------------檔案類 Function ---------
#---檢查檔案編碼
def fileencoding(filename):
    rawdata=open(filename,"rb").read(10000)
    charenc=chardet.detect(rawdata)
    return(charenc["encoding"])

    
# --- 檢查檔案Size
def getfilesize(filename):
    return(os.path.getsize(filename))
    

#------------------網頁類 Function ---------    
# --- 下載檔案    
def downloadfile(url,filename,ldap_auth=False):
    if ldap_auth==True:
        domain=read_from_config(keyname='ldap_domain',default_val='')
        userid=read_from_config(keyname='ldap_userid',default_val='')
        userpwd=read_from_config(keyname='ldap_password',default_val='',ispassword=True)

        #---如果沒有認證資訊則詢問        
        if domain=='' or userid=='' or userpwd=='':
            print("--- Please provide Autho information ---")
            domain=input("Please enter domain Name:")
            userid=input("Please enter LDAP User ID:")
            userpwd=input("Please enter LDAP User Password:")
            
            if domain=='' or userid=='' or userpwd=='':
                print("No Auth Information , Download Fail!!!")
                return(False)
            else:
                domain=read_from_config(keyname='ldap_domain',default_val=domain,write_back=True)
                userid=read_from_config(keyname='ldap_userid',default_val=userid,write_back=True)
                userpwd=read_from_config(keyname='ldap_password',default_val=userpwd,write_back=True,ispassword=True)
                print("---寫入參數檔-----")
        #---- 進行認證及下載                
        authstr=domain+'\\'+userid
        r=requests.get(url,auth=HttpNtlmAuth(authstr,userpwd), stream=True)    
        
    else:
        r=requests.get(url, stream=True)    
    
    #local_filename = url.split('/')[-1]    
    print("Status:",r.status_code)
    if r.status_code ==200:
        with open(filename, 'wb') as f:
            shutil.copyfileobj(r.raw, f)
        print("DownLoad ", url, " to ",filename , " success!!")
        return(True)
    else:
        print("DownLoad ", url, " Fail, ResponseCode:=",r.status_code)
        return(False)


#------------------加解密類 Function ---------    
# --- 加密
def enCodePasswd(passwd):
    # 加鹽
    so=random.choice('abcdefghijklmnopqrstuvwxyz') +random.choice('abcdefghijklmnopqrstuvwxyz')
    a=so+str(base64.b64encode(passwd.encode("utf-8")),'utf-8')
    return(str(base64.b64encode(a.encode("utf-8")),'utf-8'))

# --- 解密    
def deCodePasswd(passwd):
    a=base64.b64decode(passwd).decode("utf-8")
    # 去鹽
    return(base64.b64decode(a[2:]).decode("utf-8"))
