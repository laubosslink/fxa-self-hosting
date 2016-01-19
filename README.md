# fxa-self-hosting
Instructions for hosting a Firefox Accounts instance on your own domain name

## Work in progress

Do *not* use this in production! It's not ready yet. :) Most services still use the development settings, so that's entirely insecure. Apart from that, all it can do so far is show you an `Unexpected error` message (still debugging that).

I'm working on the notes.txt file in this repo to try to get this working on a Macbook with Docker, behind a self-hosted pagekite proxy.

# Get a LetsEncrypt certificate

````bash
docker run -it --net=host --rm -v `pwd`/letsencrypt:/etc/letsencrypt fxa-letsencrypt /bin/bash

$ service apache2 start
 -> check http://fxa.michielbdejong.com and https://fxa.michielbdejong.com (cert warning) show the apache default page
$ ./letsencrypt-auto --apache --text -vv
 -> answer the questions: Yes, fxa.michielbdejong.com, michiel@mozilla.com, Agree
$ exit

cp -r `pwd`/letsencrypt/live/fxa.michielbdejong.com `pwd`/fxa-cert
chmod -R ugo+r `pwd`/fxa-cert
cat `pwd`/fxa-cert/cert.pem `pwd`/fxa-cert/chain.pem > `pwd`/fxa-cert/combined.pem
````

# Set up your pagekite frontend

Replace 'secretsecretsecret' with the secret from your ~/.pagekite.rc file in the
following command, and run it on the pagekite frontend (the server to which DNS
for fxa.michielbdejong.com points):

````bash
pagekite.py --isfrontend --domain *:fxa.michielbdejong.com:secretsecretsecret --ports=80,1111,3030,5000,8000,443,9010
echo TODO: not use a http connection to the frontend
````

# Run setup.sh

Check the final output, maybe restart pagekite frontend and backend
(killing all pagekite processes from `ps auxwww | grep pagekite` in between)
until there are no rejected duplicates https://fxa.michielbdejong.com looks
the same as https://192.186.99.100 (or whatever your Docker VM IP)

If DNS hasn't propagated yet, you may need something like:

docker exec -u root -it verifier.local /bin/bash
docker exec -u root -it profile /bin/bash
-> echo 45.32.232.152 fxa.michielbdejong.com >> /etc/hosts

# Creating your account

Sign up on https://fxa.michielbdejong.com:3030/, and instead of going to look
for the verification email, run:

 docker exec -it httpdb mysql -e "USE fxa; UPDATE accounts SET emailVerified=1;"

to mark your email address as verified.

NB: If you get https://fxa.michielbdejong.com:3030/unexpected_error, run
localStorage.clear() in the console and hard-refresh.

# Configuring syncto (only necessary for Firefox OS users)

Looking for a proper way to do this through env vars; until then, find the
syncto container (b5c1ba63de07 in this example) and:

docker exec -it -u root b5c1ba63de07 /bin/bash
root@b5c1ba63de07:~# apt-get update && apt-get install -yq vim
root@b5c1ba63de07:~# vim /app/.venv/lib/python2.7/site-packages/syncclient/client.py

And then apply the changes as per
https://github.com/michielbdejong/syncclient/commit/25e4e98911ba19833d624a5a3a873bae2aa3f7e1
and restart the syncto and fxa-self-hosting containers (f5446f37393d in this example):

docker restart b5c1ba63de07
docker restart f5446f37393d

# Configuring syncserver

Similarly, edit the syncserver:

root@f90bcddd85fa:/home/app/syncserver# vim ./local/lib/python2.7/site-packages/tokenserver/verifiers.py
-> edit verifier_url = "http://verifier.local:5050/v2"

# Debugging

To debug one of the containers, e.g. the one with container id ea298056cc in `docker ps`:

````bash
 docker exec -u root -it ea298056cc /bin/bash
 # add some console.log statements to the code
 docker restart ea298056cc
 docker logs -f ea298056cc
````

You can also run the container interactively, check setup.sh for the startup params for each one.

You may also have to restart containers that link to the restarted one, for instance
the main fxa-self-hosting proxy.
