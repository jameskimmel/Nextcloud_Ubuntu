# Nextcloud_Ubuntu
This is a collection of 4 tutorials, on how to install Nextcloud on Ubuntu.

The goal of each of these guides, is to have **no warnings** in the admin center and the instance should get a **perfect security score** from the nextcloud [security scanner](https://scan.nextcloud.com).

There are tutorials that use "bare metal" installations (classical LAMP stack) or ones that use Docker Compose. There are also versions behind a NGINX reverse proxy.

They are step by step instructions, aimed at beginners. They start with quiet difficult topics like network requirements and split DNS, since that knowledge is IMHO needed beforehand. 
They are **not** guides on how to run Nextcloud without certs, maybe only locally or over a VPN. IMHO it is so easy to get a valid cert for Nextcloud, there is no reason why should go without a valid cert. 

|Tutorial | Bare metal | Docker | behind NGINX reverse proxy|
|--- | --- | --- | ---|
|[nextcloud.md](https://github.com/jameskimmel/Nextcloud_Ubuntu/blob/main/nextcloud.md)   | :white_check_mark: | :no_entry_sign: | :no_entry_sign:|
|[nextcloud_behind_NGINX_proxy.md](https://github.com/jameskimmel/Nextcloud_Ubuntu/blob/main/nextcloud_behind_NGINX_proxy.md)  | :white_check_mark: | :no_entry_sign: | :white_check_mark:|
|[nextcloud_AIO_Docker_Compose.md](https://github.com/jameskimmel/Nextcloud_Ubuntu/blob/main/nextcloud_AIO_Docker_Compose.md) | :no_entry_sign: | :white_check_mark: | :no_entry_sign:|
|[nextcloud_AIO_Docker_Compose_behind_NGINX_proxy.md](https://github.com/jameskimmel/Nextcloud_Ubuntu/blob/main/nextcloud_AIO_Docker_Compose_behind_NGINX_proxy.md) | :no_entry_sign: | :white_check_mark: | :white_check_mark:|

Feel free to offer feedback, or open up a PR :slightly_smiling_face:
