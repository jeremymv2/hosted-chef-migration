# hosted-chef-migration
Migrate your data off of Hosted Chef to a private Chef server

## Assumptions
There are various assumptions made here, please carefully take these into account as this may not be a one-size-fits-all.

* This is migrating Basic data only (cookbooks, environments, roles, data_bags, nodes, clients)
* You utilize an admin user with full privileges on both Chef servers
* The end goal is to only have to change `chef_server_url` in each node's `client.rb` - keeping existing client certificates.

## Set up a monolythic repo for the migration

```
chef generate repo migration-repo
```

## Create a knife.rb for SRC and DST

Reference an admin user with full privileges.

```
cd migration-repo

cat <<EOF> knife_src_server.rb
current_dir = File.dirname(__FILE__)
chef_server_url          "https://manage.chef.io/organizations/jeremyinc"
node_name                "jmillerv2"
client_key               "#{current_dir}/jmillerv2.pem"
versioned_cookbooks true

EOF

cat <<EOF> knife_dst_server.rb
current_dir = File.dirname(__FILE__)
chef_server_url          "https://my.company.com/organizations/lob1"
node_name                "jmex2"
client_key               "#{current_dir}/jmex2.pem"
versioned_cookbooks true

EOF
```


## Download ALL the objects from Managed Chef Server

Take care to rerefernce the `chef-repo-path` created above.
```
knife download --chef-repo-path ~/Devel/ChefProject/migration-repo -c knife_src_server.rb /
```

## Upload the objects to your private Chef Server

```
knife upload --chef-repo-path ~/Devel/ChefProject/migration-repo -c knife_dst_server.rb /cookbooks
knife upload --chef-repo-path ~/Devel/ChefProject/migration-repo -c knife_dst_server.rb /data_bags
knife upload --chef-repo-path ~/Devel/ChefProject/migration-repo -c knife_dst_server.rb /environments
knife upload --chef-repo-path ~/Devel/ChefProject/migration-repo -c knife_dst_server.rb /roles
knife upload --chef-repo-path ~/Devel/ChefProject/migration-repo -c knife_dst_server.rb /nodes
knife upload --chef-repo-path ~/Devel/ChefProject/migration-repo -c knife_dst_server.rb /clients
```

## Change the chef_server_url

At this stage, you should have all the base data on the target Chef Server.
Take one node from your fleet that you are comfortable testing with and change `chef_server_url` in the `client.rb` file
pointing it a the new target Chef Server.  

Trigger a `chef-client` run on that node - it should converge without issue.
If it converges, make the `client.rb` change on the rest of your fleet in stages - ensuring that nodes are checking in successfully.
