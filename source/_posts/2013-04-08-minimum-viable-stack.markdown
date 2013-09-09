---
layout: post
title: "minimum viable stack"
date: 2013-04-08 14:55
comments: true
categories: 
---

After [migrating to my own VPS solution from Heroku][16] at [Responsa][0], I've traveled across the internet land in order to select and build a solid production stack.
Now that I think I've reached it I want to share my choices in case anyone has to go through the same process.

## Service selection criterion

I'm using Chef as provisioning tool so while looking around for services I immediately run the query on Google: "#{service} cookbook". Usually there is always a cookbook, but most of them are really bad. However if the cookbook is good the chances of that service to be picked are very high.

## Grocery list

The list of things I need boils down to this:

- DB and cache
- Ruby
- Monitoring service for CPU, memory, IO etc…
- Service manager
- Logs aggregator
- Backups manager
- Web server

### DB

[MongoHQ][2] has a very good service with awesome customer support. Can't recommend them enough.

For session and cache storage I chose [Redis][3] and [Memcached][4] respectively.

I've yet to find a good cookbook for Redis 2.6. I guess I'll have to upgrade the cookbook myself when I have the chance.

### Ruby

[rbenv][5] and [ruby_build][6] are the killer combo. Just install your Ruby version as global. After all who needs multiple versions of ruby on the production machine?

### Services manager

I'm not really opinionated about service managers so this was the most community driven choice. Many are using Runit and the [cookbook][1] is pretty good.

### Monitoring

I've tried Munin but it felt like using a walkman in 2013.
Hard to configure with chef solo (no need for server and client in that case) and badly documented.

New Relic kicks ass. The [cookbook][7] is one include_recipe away from running and the dashboard is feature rich and easy to use. The downside is that if you have many instances the price might go out of bounds…

### Logs aggregator

There are some nice competitors in this space. [Loggly][8], [Logentries][9] and [Papertrail][10]. Again this was a decision driven by the quality of the cookbook and Papertrail has a [pretty good one][11].

I use rsyslog to put all my important logs in one place, Papertrail grabs them and cleverly separates services like Unicorn, sshd, Nginx etc…
The search feature is also useful.

### Backups

[Whenever][12] + [Backup][13] are the killer combo. I've set them up with a custom made cookbook that updates the crontab with whenever at every chef run.

### Web server

[Nginx][17]. Boom. Done.

### Bonus

I don't like when I close my laptop and after reopening it ssh hangs. To solve that issue I use [mosh][14] which needs a server installed on the machine. Luckily there is a [cookbook][15] also for that.

You're welcome to share your stack in the comments below.

[0]: https://goresponsa.com
[1]: http://community.opscode.com/cookbooks/runit
[2]: https://www.mongohq.com/home
[3]: https://github.com/FunGoStudios/redis-cookbook
[4]: http://community.opscode.com/cookbooks/memcached
[5]: https://github.com/fnichol/chef-rbenv
[6]: https://github.com/fnichol/chef-ruby_build
[7]: http://community.opscode.com/cookbooks/newrelic
[8]: http://loggly.com/
[9]: https://logentries.com/
[10]: https://papertrailapp.com/
[11]: https://github.com/responsa/papertrail-cookbook
[12]: https://github.com/javan/whenever
[13]: https://github.com/meskyanichi/backup
[14]: http://mosh.mit.edu/
[15]: http://community.opscode.com/cookbooks/mosh
[16]: http://matteodepalo.github.io/blog/2013/03/07/how-i-migrated-from-heroku-to-digital-ocean-with-chef-and-capistrano/
[17]: http://community.opscode.com/cookbooks/nginx