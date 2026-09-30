# Imagem NetBox com a identidade visual da Direct e a configuração deste repositório embutidas.
ARG NETBOX_IMAGE=docker.io/netboxcommunity/netbox:v4.7-5.1.1

# Gera favicon (.ico) e apple-touch-icon (.png) a partir do SVG
FROM docker.io/library/alpine:3.22 AS icons
RUN apk add --no-cache rsvg-convert imagemagick
COPY src/img/direct-icon.svg /src/icon.svg
COPY src/img/direct-light.svg /src/logo.svg
RUN mkdir /out \
    # Variante para o tema escuro: texto azul-marinho (#13367f) vira branco
    && sed 's/#13367f/#ffffff/Ig' /src/logo.svg > /out/logo-dark-theme.svg \
    && for s in 16 32 48 64; do rsvg-convert -w "$s" -h "$s" --keep-aspect-ratio /src/icon.svg -o "/tmp/icon-$s.png"; done \
    && magick /tmp/icon-16.png /tmp/icon-32.png /tmp/icon-48.png /tmp/icon-64.png /out/netbox.ico \
    && rsvg-convert -w 180 -h 180 --keep-aspect-ratio /src/icon.svg -o /tmp/touch.png \
    && magick /tmp/touch.png -gravity center -background white -extent 180x180 /out/netbox_touch-icon-180.png

FROM ${NETBOX_IMAGE}

COPY configuration/ /etc/netbox/config/

# Título da aba do navegador: "<página> | NetBox - Direct"
RUN sed -i 's/{% endblock %} | NetBox</{% endblock %} | NetBox - Direct</' /opt/netbox/netbox/templates/base/base.html \
    && grep -q '| NetBox - Direct</title>' /opt/netbox/netbox/templates/base/base.html

# O collectstatic já rodou na imagem base: substitui tanto a origem (project-static/img)
# quanto os arquivos servidos pelo Granian (static/)
# logo_netbox_dark_teal   -> exibido no tema claro
# logo_netbox_bright_teal -> exibido no tema escuro
COPY src/img/direct-light.svg /opt/netbox/netbox/static/logo_netbox_dark_teal.svg
COPY --from=icons /out/logo-dark-theme.svg /opt/netbox/netbox/static/logo_netbox_bright_teal.svg
COPY src/img/direct-light.svg /opt/netbox/netbox/project-static/img/logo_netbox_dark_teal.svg
COPY --from=icons /out/logo-dark-theme.svg /opt/netbox/netbox/project-static/img/logo_netbox_bright_teal.svg
COPY --from=icons /out/netbox.ico /opt/netbox/netbox/static/netbox.ico
COPY --from=icons /out/netbox.ico /opt/netbox/netbox/project-static/img/netbox.ico
COPY --from=icons /out/netbox_touch-icon-180.png /opt/netbox/netbox/static/netbox_touch-icon-180.png
COPY --from=icons /out/netbox_touch-icon-180.png /opt/netbox/netbox/project-static/img/netbox_touch-icon-180.png
