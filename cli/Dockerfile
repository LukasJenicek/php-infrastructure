FROM php:7.3-alpine

ARG UID=1000
ARG GID=1000
ARG USER=appuser

RUN apk update && \
    apk add --virtual build-deps python-dev musl-dev curl unzip $PHPIZE_DEPS bash bash-doc bash-completion vim zsh git

RUN curl -sS https://getcomposer.org/installer -o composer-setup.php && \
    php composer-setup.php --install-dir=/usr/local/bin --filename=composer && \
    rm composer-setup.php

RUN addgroup -S -g $UID $USER && adduser -u $GID -S $USER -G $USER

RUN pecl install xdebug && docker-php-ext-enable xdebug

COPY config/* $PHP_INI_DIR/conf.d/

USER $USER

RUN wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh || true

#COPY bash/.bashrc '/home/appuser/.bashrc'
COPY zsh/.zshrc '/home/appuser/.zshrc'

WORKDIR app/
CMD ["zsh"]
