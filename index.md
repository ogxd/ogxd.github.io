# OGXD Website

Welcome !

{% assign public_repositories = site.github.public_repositories | where:'fork', false | sort: 'stargazers_count' | reverse %}
{% for repository in public_repositories limit: 9 %}
## {{ repository.name }}
{{ repository.description }}  
`{{ repository.language }}` | ⭐ {{ repository.stargazers_count }} | 👁 {{ repository.watchers_count }}  
[Source]({{ repository.html_url }}) | [Release]({{ repository.latest_release }})
{% endfor %}