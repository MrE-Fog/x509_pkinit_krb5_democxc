FROM centos:7
RUN yum -y install deltarpm && \
    yum -y clean all && \
    yum -y install krb5-server krb5-libs krb5-workstation krb5-pkinit && \
    yum -y install less which openssl bind-utils tree mlocate iproute tcpdump

# configure KDC

COPY krb5.conf /etc/krb5.conf
COPY krb5.conf /var/kerberos/krb5kdc/kdc.conf
RUN useradd -m user1
RUN useradd -m user2
RUN kdb5_util create -s -r EXAMPLE.COM -P abc123
RUN kadmin.local -q 'add_principal +requires_preauth -nokey user1'
#RUN kadmin.local -q "addprinc -pw xyz456 user1@EXAMPLE.COM"

# create CA root certificates
# https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/
RUN mkdir -p /root/tiny_ca  && \
    openssl genrsa -out /root/tiny_ca/cakey.pem 2048 && \
    openssl req -key /root/tiny_ca/cakey.pem -new -x509  \
                -out /root/tiny_ca/cacert.pem -days 365 \
                -subj '/C=US/ST=California/L=Modesto/O=X510_ROOT_TEST_CERT/OU=DO NOT USE IN PRODUCTION - DO NOT TRUST/CN=kdc.x509_pkinit_krb5_demo_default' 

# create KDC x509 cert
RUN mkdir -p /var/tmp/certs/kdc_certs 
COPY kdc_extensions.ssl /var/tmp/certs/kdc_certs
RUN cp /root/tiny_ca/cacert.pem /var/tmp/certs

RUN openssl genrsa -out /var/tmp/certs/kdc_certs/kdckey.pem 2048 && \
    openssl req -new -out /var/tmp/certs/kdc_certs/kdc.req \
                -key  /var/tmp/certs/kdc_certs/kdckey.pem \
                -subj '/C=US/ST=California/L=Modesto/O=KDC_X509_CERT/OU=DO NOT USE IN PRODUCTION/CN=kdc.x509_pkinit_krb5_demo_default' && \
    openssl x509 -req -in /var/tmp/certs/kdc_certs/kdc.req \
                 -CAkey /root/tiny_ca/cakey.pem -CA /root/tiny_ca/cacert.pem \
                 -out /var/tmp/certs/kdc_certs/kdc.pem -days 365 \
                 -extfile  /var/tmp/certs/kdc_certs/kdc_extensions.ssl \
                 -extensions  kdc_cert  \
                 -CAcreateserial 
                
# create user1 x509 cert
RUN mkdir -p /var/tmp/certs/user1_certs 
COPY client_extensions.ssl /var/tmp/certs/user1_certs

RUN openssl genrsa -out /var/tmp/certs/user1_certs/clientkey.pem 2048 && \
    openssl req -new -key /var/tmp/certs/user1_certs/clientkey.pem \
                -out /var/tmp/certs/user1_certs/client.req \
                -subj '/C=US/ST=California/L=Modesto/O=user1_X509_CERT/OU=DO NOT USE IN PRODUCTION/CN=kdc.x509_pkinit_krb5_demo_krb5_demo' && \
    openssl x509 -CAkey /root/tiny_ca/cakey.pem -CA /root/tiny_ca/cacert.pem \
                 -req -in /var/tmp/certs/user1_certs/client.req \
                 -extfile /var/tmp/certs/user1_certs/client_extensions.ssl \
                 -extensions client_cert \
                 -days 365 -out /var/tmp/certs/user1_certs/client.pem   \
                 -CAcreateserial 

# start container
CMD ["/usr/sbin/krb5kdc", "-n", "-w", "10"]
