How do I Upgrade my Mastodon Docker Deployment?

I deployed Mastodon from the Linode marketplace and a new version with important security patches was just released. The documentation for upgrading a docker deployment on the [Mastodon GitHub](https://github.com/mastodon/mastodon/releases/tag/v4.1.4) page is minimal and I wasn't able to get it to work. 

---

The Marketplace app is set to pull the latest Mastodon version at the time of deployment. That means you will need to upgrade to any new releases manually. By following these steps, hopefully the process will be relatively straightforward. It's worth noting I upgraded from v4.1.2 to v4.1.4 using this process.

First, ssh into your server and ensure your system is up to date:
```
sudo apt update && sudo apt upgrade
```

# Install Docker Engine
For some reason, the version of docker that's installed on the Marketplace Mastodon deployment will not upgrade the version out of the box. If you don't reinstall the Docker Engine, you'll get an error trying to build the new services. Follow the installation instructions found in the [Docker Documention](https://docs.docker.com/engine/install/debian/). Without the `docker-buildx-plugin` this update won't work. 

Check to make sure the buildx plugin was installed:

```
docker buildx version
github.com/docker/buildx v0.11.1 b4df085
```

# Backup your Database
Next, you want to create a backup of your database. Switch to the `mastodon` user and, change to the `/home/mastodon/live` directory and find the database container. It should be named `live_db_1`. 
- su mastodon
- cd /home/mastodon/live
- sudo docker ps -a 

Use this command to connect to create a backup of your database on the container, changing <date> to the date you creating the backup for tracking purposes:
```
sudo docker exec -it live_db_1 /bin/bash -c 'pg_dumpall -U mastodon -f db_backup-<date>.sql && exit'
```
Copy the backup to your host:
```
sudo docker cp live_db_1:/db_backup-<date>.sql /home/mastodon/
```
If the dump was successful, the `tail` output of your `db_backup-<date>.sql` will look like this:
```
SET row_security = off;

--
-- PostgreSQL database dump complete
--

--
-- PostgreSQL database cluster dump complete
--
```

# Upgrade to new release

Now it's time to upgrade your containers to the newest release using a series of `git` and `docker` commands from the `/home/mastodon/live` directory. 

Fetch the version tags from the remote and stash the changes to your working directory.
```
git fetch --tags
sudo git stash
```

Use this command to switch to the latest version:
```
git checkout v4.1.4
```

You will likely see some errors here but they'll be cleaned up in the next step. 
Check for modified files in the working directory
```
git stash pop
```

Use the following command to discard any changes you made to files in the working directory:
```
sudo git restore <modified files>
```
Once you have discarded the changes that showed up when you initially ran `git stash pop`, run it again to ensure no other files need to be restored. 

Build the Web, Streaming and Sidekiq services
```
sudo docker compose build
```
Check for recently created images - you should see the image you just created at the top:
```
docker images
```
Run the pre-deployment database migrations:
```
sudo docker compose run --rm -e SKIP_POST_DEPLOYMENT_MIGRATIONS=true web rails db:migrate
```
Check the success of the migration with `echo $?`. If the output is `0` then the migration was successful. 

Shutdown all docker-compose services
```
sudo docker compose down
```
Clear the cache:
```
sudo docker compose run --rm web bin/tootctl cache clear
```

Run post-deployment database:
```
sudo docker compose run --rm web rails db:migrate
```
Once again, if the output of `echo $?` is `0`, the migration was successful. 

Create and start the docker containers:
```
sudo docker compose up -d
```

Remove any unused docker data. This will get rid of any images or containers not in use with your mastodon deployment so use with caution:
```
docker system prune -a 
```

Special thanks to Floris Van den Abeele and their work in this post titled [Upgrading your mastodon instance](https://vdna.be/site/index.php/2021/03/upgrading-a-mastodon-instance/), where I got most of the info for this guide. 
