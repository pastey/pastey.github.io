+++
title = 'Remove a Git tag. Mission impossible'
date = '2024-02-11'
draft = false
+++

Sure, you can do `git tag -d <tagname>` and then `git push --delete origin <tagname>`. But chances are high that the tag will resurrect soon. Keep reading if you want to know more.

<!--more-->

To translate our app, we use a third-party service. The process of getting localizable strings from our code to the service, and translations back to the code is fully automated.

Recently, we found out that the upload to the service is not atomic. This means that, for example, 6 out of 8 languages may upload to the server, leaving our translations inconsistent.

We had to somehow make uploads atomic. We decided to use a kind of Write Ahead Lock (WAL).

As we are operating in a Git repo, we thought that using tags as a transaction marker would be a good option. The operation would work as follows:
 - create a tag with a special name
 - run the operation
 - delete the tag

If anything goes wrong during the operation (internal server error, network goes down, etc.), the tag will not be deleted. If on the next launch we find that the tag exists, the restoration actions will take place.

To be super safe, we decided to push those special tags.

Next few days, we started observing mysterious behavior. The restoration mechanism worked too often. But once we checked the logs, the previous run did not fail! The tag was removed properly by the script. The only way for the tag to reappear was if a human recreated it. And turns out, they did! But why?

Upload to the service takes some time (several minutes). During this period, people can get the tag. And then come two things:
 - very few people prune tags. To be frank, I never prune tags myself. For example, SourceTree has been gathering interest for this feature since 2017 (must be a very complex feature, I guess).
 - GUI apps push tags by default (I checked SourceTree and Fork)

So I removed tag push. And to be sure, added `--no-tags` to the `git fetch` so that tags are not fetched.

Done! Done? Not so fast! It's Git, babe...

Transactions for the already pushed tags kept on rolling back and forth. Tags somehow kept on coming back to life from remote.

The whole fetch command looked like this: `git fetch --no-tags -Pp`.

`-P` means "prune tags".

And `--no-tags` says "do not fetch new tags". I thought. Git has a different idea. `-P` effectively disables `--no-tags` and fetches all tags to... prune the deleted ones afterwards.

I removed the `-P` and then finally our transactions calmed down.

Overall. If you clean up tags, remember that they can come back to life easily. Both CLI and especially GUI clients make this issue highly probable.
