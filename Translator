#!/usr/bin/python
# -*- coding: utf-8 -*-

#Import the Natural Language Toolkit
import nltk
#Library to connect to MySQL from Python
import MySQLdb
#Library to run command line arguments from Python
import subprocess
#Library for reading command-line input
import sys

#connect to the database
class DB:
  conn = None
  #self is the object of the class DB, used like "this" in Java
  def connect(self):
    #Connect to the local server and MySQL database
    self.conn = MySQLdb.connect('localhost', 'root', 'root', 'dict', charset="utf8")

  #Execute the "sql" query
  def query(self, sql):
    try:
            #The cursor is used to execute the query
      cursor = self.conn.cursor()
      cursor.execute(sql)

      #In case connection wasn't established, repeat the process
    except (AttributeError, MySQLdb.OperationalError):
      self.connect()
      cursor = self.conn.cursor()
      cursor.execute(sql)
    return cursor

def userInput():
    #Video file and Subtitle file names are global, to be used during input and output
    global vidFile, subFile
    
    #User enters the name of the video file
    #vidFile = raw_input('Enter a file name: ') #Need to add GUI
    vidFile = sys.argv[1]
    
    #Open the video's .srt file (SRT file has the same name as the video)
    index = vidFile.find('.')
    subFile = vidFile[:index]+".srt"

    #Play video with English subtitles as a preview
    playVideo("Purisa")

    #Open the subtitle file so that one line can be translated at a time
    f = open(subFile, 'r')
    return f

def playVideo(font):
    #Play video on VLC Player with in the given font
    global vidFile, subFile
    font = "--freetype-font="+font
    #Call VLC from command-line
    p = subprocess.Popen(["/usr/bin/vlc",vidFile,font])
    p.wait()
  
def lookup(eng,count):
    ##print "Entered lookup for ",eng," Count: ",count
    #Connect to the database
    db = DB()
    #Convert int to string to create the SQL query
    count = str(count)
    sql = "SELECT * FROM dictionary where English='"+eng+"' and Count='"+count+"'"
    #Query the database
    cursor = db.query(sql)
    #Fetch the relevant row
    row = cursor.fetchone()
    #In case no data has been fetched, try again with the default Count value (0)
    if row == None:
      count = "0"
      sql = "SELECT * FROM dictionary where English='"+eng+"' and Count='"+count+"'"
      cursor = db.query(sql)
      row = cursor.fetchone()
    ##print "Query: ",sql
    ##print "returned: ",row[1]," ",row[3]
    #Return the Hindi word along with the Count
    return row[1],row[3]

def lookupSubj(eng):
    ##print "Entered subject lookup for ",eng
    #Connect to the database
    db = DB()
    #Frame the SQL query
    sql = "SELECT * FROM dictionary where English='"+eng+"'"
    #Query the database
    cursor = db.query(sql)
    #Fetch the relevant row
    row = cursor.fetchone()
    ##print "Query: ",sql
    ##print "returned: ",row[1]," ",row[3]
    #Return the Hindi word and the Count
    return row[1],row[3]


def translate(tagged):

    #Question Flag
    flagQ = 0
    #Verb Flag
    flagVerb = 0
    #Flag for the special case of "dont"
    flagDont = 0
    #Flag for the special case for a "can" question
    flagCan = 0
    
    prevCount = 0
    
    subject = ""
    obj = ""
    verb = ""
    translatedSent = ""
    punctuation = ""
    count = 0
    index = 0
    indexStart = -1
    indexEnd = -1

    for word in tagged:
        #"The" does not exist in Hindi
        if word[0] == "the":
            tagged.pop(index)
        #Look for extra punctuation
        if word[0] == "." or word[0] == "?":
            punctuation = word[0]
            tagged = tagged[:-1]
            break
        if "WP" in word or "WRB" in word:
            flagQ = 1
        if word[1][:2] == "VB":
            if indexStart == -1:
                indexStart = index
                ##print "Start: ",indexStart
        elif indexStart > -1 and flagVerb == 0:
            indexEnd = index
            flagVerb = 1
            ##print "End: ",indexEnd
        index = index + 1

    if flagQ == 1:
      o = tagged[:indexStart]
      v = tagged[indexStart:indexEnd]
      s = tagged[indexEnd:]
    else:
      s = tagged[:indexStart]
      v = tagged[indexStart:indexEnd]
      o = tagged[indexEnd:]

    temp = []
    #If no verb has been identified, it must have been mis-tagged as an object
    if v == [] and len(tagged) > 2:
      for word in o:
        if word[0] == "dont":
          flagDont = 1
          v.insert(0,("do","VBP"))
          temp.insert(0,("not","RB"))
          continue
        temp.append(word)
      o = temp

    index = 0
    for word in s:
      if word[0] == "can":
        s.pop(index)
        s.insert(index,("what","POS"))
        v.append(("can","POS"))
        break
      index = index + 1

    index = 0
    for word in o:
      if word[0] == "with":
        o.pop(index+1)
        break
      index = index + 1
        
    ##print "Subject: ",s
    ##print "Verb: ",v
    ##print "Object: ",o
    
    for word in s:
      #Since the subject sets the count, the query should not have a "Count" parameter
      trans,count = lookupSubj(word[0])
      if count < prevCount:
        count = prevCount
        prevCount = count
      else:
        subject = subject + trans + " "

    ##print "Translated Subject: ",subject
    temp = v
    for word in v:
      if "VBG" in word:
        index = v.index(word)
        temp[index], temp[index-1] = temp[index-1], temp[index]

    v = temp
    for word in v:
      ##print word[0]
      trans = lookup(word[0], count)[0]
      verb = verb + trans + " "

    ##print "Translated Verb: ",verb
    
    for word in o:
      trans = lookup(word[0], count)[0]
      obj = obj + trans + " "

    ##print "Translated Object: ",obj

    translated = subject + obj + verb + punctuation
    return translated

def checkContraction(sentence,word,num):
  flagContraction = sentence.find(word)
  if flagContraction >= 0:
    remove = word.find("'")
    word = word[:remove] + word[remove+1:]
    sentence = sentence[:flagContraction] + word + sentence[flagContraction+num:]
  return sentence

def findPunc(s,x):
  index = s.find(".",x)
  temp = index
  index = s.find("?",x)
  if index != -1 and temp != -1:
    if index > temp:
      index = temp
  elif index == -1:
    if temp != -1:
      index = temp
  return index

def preproc(sentence):
  #Bring the entire sentence into lowercase
  sentence = sentence.lower()
    
  #Look for contractions
  sentence = checkContraction(sentence,"don't",5)
  sentence = checkContraction(sentence,"isn't",5)
  flagIts = sentence.find("it's")
  ###print "flagIts: ",flagIts
  if flagIts >= 0:
    sentence = sentence[:flagIts] + "it is" + sentence[flagIts+4:]
  ##print "\nSENTENCE: ",sentence.upper()

  #Tokenize the sentence to extract words and punctuation
  tokens = nltk.word_tokenize(sentence)

  #Tag the extracted tokens using a Part-of-Speech (POS) Tagger
  tagged = nltk.pos_tag(tokens)

  return tagged

def translatedSub():
    fSub = userInput() #English subtitles file
    fTrans = open("temp.srt",'w') #temporary subtitles file
    sentence = "Start"
    translated = ""
    tagged = ""
    #The three flags denote the identification of the Subject, Verb and Object

    i = 0
    for line in fSub:
      fTrans.write(line) #Frame number
      line = fSub.next() #Time stamp 00:00:00 ---> etc
      fTrans.write(line)
      line = fSub.next() #First line
      ##print line
      copy = line
      #Reading the entire subtitle in a single frame
      while copy.strip() != "":
        #For music
        if sentence[0] == "[":
            length = len(sentence)-2
            sentence = sentence[1:length]
            sentence = lookup(sentence,0)
            translated = "[" + sentence + "]"
            
        else: #Ordinary sentences
          indexPrev = 0
          indexPunc = findPunc(line,indexPrev) #End of first sentence
          #Segment out each subsequent sentence based on punctuation
          while indexPunc != -1:
            sentence = line[indexPrev:indexPunc+1]
            #check contractions, lowercase, POS tagging
            tagged = preproc(sentence)
            ##print tagged
            #This is where the translator kicks in
            translated = translate(tagged)
            ##print translated
            indexPrev = indexPunc + 1
            indexPunc = findPunc(line,indexPrev) #isolate each sentence in line
            ##print "Translated: ",translated
            fTrans.write(translated.encode("UTF-8"))
          fTrans.write("\n")
          line = fSub.next() #First line
          copy = line
      fTrans.write("\n\n")
      ##print i
      
      if i == 5:
        break
      i = i + 1
      
    global subFile
    
    #Make temp the SRT file for the video
    index = subFile.rfind("/")
    origFile = subFile[:index+1]+"orig"+subFile[index+1:]
    subprocess.call(["mv",subFile,origFile]) #make a copy of the English subtitle file
    subprocess.call(["mv","temp.srt",subFile]) # Make the translated file the video's SRT
    

translatedSub()
playVideo("Lohit Hindi")
##print ".srt successfully created!"
