# OpenMage PHP Container

# PHP Releases:
https://www.php.net/releases/

# Tideways Releases:
https://tideways.com/profiler/downloads

# Imagick Releases:
https://github.com/Imagick/imagick/tags

# NewRelic PHP Releases:
https://github.com/newrelic/newrelic-php-agent/releases

# Sample building for testing:
docker build -f src/7.4/src/Dockerfile -t docker.io/openmage/php:7.4.29 src/7.4/src/

# Sample building for multi arch testing and push to docker hub:
cd src/8.3/src &&  docker buildx create --use &&  docker buildx build --progress=plain --platform linux/amd64,linux/arm64 --push -t docker.io/openmage/php:8.3.0 .
