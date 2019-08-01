FROM centos:7
RUN yum -y install deltarpm && \
    yum -y clean all && \
    yum -y install krb5-server krb5-libs krb5-workstation && \
    yum -y install less which openssl

#Optional BS Packages to RM later
RUN yum install -y tree

# configure KDC

COPY krb5.conf /etc/krb5.conf
COPY krb5.conf /var/kerberos/krb5kdc/kdc.conf
RUN useradd -m user1
RUN useradd -m user2
RUN kdb5_util create -s -r EXAMPLE.COM -P abc123
RUN kadmin.local -q "addprinc -pw xyz456 user1@EXAMPLE.COM"



# create CA root certificates
# https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/
RUN mkdir -p /root/tiny_ca  && \
    openssl genrsa -des3 -passout pass:abc123 \
                    -out /root/tiny_ca/myCA.key 2048 && \
    openssl req -x509 -passin pass:abc123 -new -nodes \
                -key /root/tiny_ca/myCA.key -sha256 -days 10000 \
                -out /root/tiny_ca/myCA.pem \
                -subj '/C=US/ST=California/L=Modesto/O=X509_TEST_ROOT_CERT/OU=DO NOT USE IN PRODUCTION/CN=kdc.example.com'

# copy root cert to location for clients to read 
# TODO: make intermediate cert
            
# create KDC x509 cert
RUN mkdir -p /var/tmp/certs/kdc_certs 
COPY kdc_extensions.ssl /var/tmp/certs/kdc_certs
RUN cp /root/tiny_ca/myCA.pem /var/tmp/certs
RUN openssl genrsa -out /var/tmp/certs/kdc_certs/kdc_x509.key 2048 && \
    openssl req -new -key /var/tmp/certs/kdc_certs/kdc_x509.key \
                -out /var/tmp/certs/kdc_certs/kdc_x509.csr \
                -subj '/C=US/ST=California/L=Modesto/O=KDC_X509_CERT/OU=DO NOT USE IN PRODUCTION/CN=kdc.example.com' && \
    openssl x509 -req -in /var/tmp/certs/kdc_certs/kdc_x509.csr \
                 -CA /root/tiny_ca/myCA.pem -CAkey /root/tiny_ca/myCA.key \
                 -CAcreateserial -out /var/tmp/certs/kdc_certs/kdc_x509.crt \
                 -days 10000 -sha256 -passin pass:abc123  \
                 -extfile /var/tmp/certs/kdc_certs/kdc_extensions.ssl \
                 -extensions kdc_cert

# create user1 x509 cert
RUN mkdir -p /var/tmp/certs/user1_certs 
COPY client_extensions.ssl /var/tmp/certs/user1_certs

RUN openssl genrsa -out /var/tmp/certs/user1_certs/user1_x509.key 2048 && \
    openssl req -new -key /var/tmp/certs/user1_certs/user1_x509.key \
                -out /var/tmp/certs/user1_certs/user1_x509.csr \
                -subj '/C=US/ST=California/L=Modesto/O=user1_X509_CERT/OU=DO NOT USE IN PRODUCTION/CN=kdc.example.com' && \
    openssl x509 -req -in /var/tmp/certs/user1_certs/user1_x509.csr \
                 -CA /root/tiny_ca/myCA.pem -CAkey /root/tiny_ca/myCA.key \
                 -CAcreateserial -out /var/tmp/certs/user1_certs/user1_x509.crt \
                 -days 10000 -sha256 -passin pass:abc123 \
                 -extfile /var/tmp/certs/user1_certs/client_extensions.ssl \
                 -extensions client_cert

# start container
CMD ["/usr/sbin/krb5kdc", "-n"]
