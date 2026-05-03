## Phase 4: Automating with Shuffle amd Genearating Notification on Discord.

On the last machine we deployed for ubuntu out of 3 we will now going to install the shuffle so let's begin by following below commands.

```bash
sudo apt update && sudo apt install -y docker.io docker-compose

sudo systemctl enable docker
sudo systemctl start docker

git clone https://github.com/Shuffle/Shuffle
cd Shuffle

sudo chown -R 1000:1000 shuffle-database  
sudo swapoff -a

docker compose up -d                      

```
After the installtion is complete, access the application by navigating to http://your-ip:3001 in your web browser.
