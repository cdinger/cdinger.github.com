---
layout: post
title: "Display git revision in your Rails application footer"
lead: "It's always nice to know which version of your code is deployed. When your
product owner discovers a bug in your staging/test instance especially, you want
to know 'what version was that?'. Or, more importantly, 'who checked in and
deployed that code?!'."
---
It's always nice to know which version of your code is deployed. When your
product owner discovers a bug in your staging/test instance especially, you want
to know "what version was that?". Or, more importantly, "who checked in and
deployed that code?!".

In our apps, we've taken to adding the git revision and Rails environment to our
application footers. We use a four-tier system (DEV, TST, ENT, and PRD) so
knowing which environment your app is running in sometimes gets confusing. Our
template system includes a partial that includes both of these bits of info:

![git revision footer screenshot](/images/git_revision_footer.png)

Each of our Rails apps include an intitializer that grabs the `REVISION` file
that capistrano creates during a deploy:

{% highlight ruby %}
# config/initializers/git_revision.rb

# Get the deployed git revision to display in the footer
module Git
  REVISION = File.exists?(File.join(Rails.root, 'REVISION')) ? File.open(File.join(Rails.root, 'REVISION'), 'r') { |f| GIT_REVISION = f.gets.chomp } : nil
  VERSION = File.exists?(File.join(Rails.root, 'VERSION')) ? File.open(File.join(Rails.root, 'VERSION'), 'r') { |f| GIT_VERSION = f.gets.chomp } : nil
end
{% endhighlight %}

If you don't use capistrano, you can always shell out to git (like we do to grab
the short SHA1):

{% highlight ruby %}
# config/initializers/git_revision.rb

# Get the deployed git revision to display in the footer
module Git
  REVISION = `SHA1=$(git rev-parse --short HEAD 2> /dev/null); if [ $SHA1 ]; then echo $SHA1; else echo 'unknown'; fi`.chomp
  VERSION = `VERSION=$(git describe --tags 2> /dev/null); if [ $VERSION ]; then echo $VERSION; else echo 'unknown'; fi`.chomp
end
{% endhighlight %}

Then you can include an application partial with content like this:

{% highlight erb %}
# app/views/shared/_version.html.erb

Revision <%= link_to Git::REVISION, "https://github.com/blah/project/commit/#{Git::REVISION}" %>@<%= Rails.env %>
{% endhighlight %}

