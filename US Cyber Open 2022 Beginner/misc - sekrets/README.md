# Sekrets
---
## Description
I didn't save the question description, but it was to the effect of '@sekretsql is passing information and we want to find out how, see his latest post'.
---
## Info Gathering 

Found @SekretSql on twitter, and there are some retweets regarding flat earth society, a tweet mentioning a steganography tool StegHide, as well as a picture of a squirrel. I downloaded the latter two, and saved the picture as sqlvaca.jfif.

![Squirrel Selfie from SekretSql!](/sqlvaca.jfif "SekretSql's Most Recent Tweet")
---
## Solution
Given the preponderance of flat earth society tweets, there is a good chance these unlock the clue in the file. I created a text file with the hashtags and usernames of the tweets, threw in a couple more variations manually and saved it to a file sktsqlpwds.txt.
```
"Flat Earth"
FlatEarth 
EarthNotAGlobe 
EarthIsFlat
flatearth
earthisflat
BoredElonMusk
KnoxLimoRental
"Flat Earth Society"
"Flat Earther"
FlatEarthSociety
FlatEarther
FlatEarthOrg
BoredErth
Earth
Flat
earth
flat
RangerUpSekrit
```

Next, I ran steghide from the command line with the passphrases above. I used Windows Powershell to automate the process.
```
PS > foreach ($line in Get-Content .\sktsqlpwds.txt) {  steghide.exe extract -p $line -sf .\sqlvaca.jfif}   
```

The extraction succeeds for the 5th term (flatearth) and wrote out flag.txt (BeSureToDrinkYourOvaltine).

## Red Herring
Not mentioned above was the several hours I spent on a sktsql webpage, trying to decode its pictures. Moral of the story is to read the question carefully, because the question mentioned the latest post only.