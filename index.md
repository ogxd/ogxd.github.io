# OGXD Website

Welcome !

{% assign public_repositories = site.github.public_repositories | where:'fork', false | sort: 'stargazers_count' | reverse %}
{% for repository in public_repositories limit: 9 %}
### {{ repository.name }}
[Github]({{ repository.html_url }})
{% endfor %}