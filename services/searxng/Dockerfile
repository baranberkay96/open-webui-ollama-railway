# Private SearXNG metasearch instance for Open WebUI's web search.
# Adds a settings.yml that enables the JSON output format Open WebUI needs.
FROM searxng/searxng:2026.10.9-f4822b3fc

COPY --chown=searxng:searxng settings.yml /etc/searxng/settings.yml

EXPOSE 8080
