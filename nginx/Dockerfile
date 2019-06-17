FROM nginx

# Delete the default configuration
RUN rm -v /etc/nginx/conf.d/default.conf

# Create a blank one using nano text editor
ADD nginx.conf  /etc/nginx/conf.d/