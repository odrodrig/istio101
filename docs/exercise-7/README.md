# Exercise 7 - Secure your services

## Mutual authentication with Transport Layer Security (mTLS)

Istio can secure the communication between microservices without requiring application code changes. Security is provided by authenticating and encrypting communication paths within the cluster. This is becoming a common security and compliance requirement. Delegating communication security to Istio (as opposed to implementing TLS in each microservice), ensures that your application will be deployed with consistent and manageable security policies.

Istio Citadel is an optional part of Istio's control plane components. When enabled, it provides each Envoy sidecar proxy with a strong (cryptographic) identity, in the form of a certificate.
Identity is based on the microservice's service account and is independent of its specific network location, such as cluster or current IP address.
Envoys then use the certificates to identify each other and establish an authenticated and encrypted communication channel between them.

Citadel is responsible for:

* Providing each service with an identity representing its role.

* Providing a common trust root to allow Envoys to validate and authenticate each other.

* Providing a key management system, automating generation, distribution, and rotation of certificates and keys.

When an application microservice connects to another microservice, the communication is redirected through the client side and server side Envoys. The end-to-end communication path is:

* Local TCP connection (i.e., `localhost`, not reaching the "wire") between the application and Envoy (client- and server-side);

* Mutually authenticated and encrypted connection between Envoy proxies.

When Envoy proxies establish a connection, they exchange and validate certificates to confirm that each is indeed connected to a valid and expected peer. The established identities can later be used as basis for policy checks (e.g., access authorization).

## Enforce mTLS between all Istio services

1. To enforce a mesh-wide authentication policy that requires mutual TLS, submit the following policy. This policy specifies that all workloads in the mesh will only accept encrypted requests using TLS.

  Apply the following `PeerAuthentication` object to enforce mTLS between the services.

  ```bash
  oc apply -f $WORK_DIR/istio101/docs/plans/guestbook-mtls.yaml
  ```

  ```yaml
  apiVersion: "security.istio.io/v1beta1"
  kind: "PeerAuthentication"
  metadata:
    name: "default"
    namespace: "istio-system"
  spec:
    mtls:
      mode: STRICT
  ```

1. Visit your guestbook application by going to it in your browser. Everything should be working as expected! To confirm mTLS is infact enabled, you can run:

  ```shell
  istioctl x describe service guestbook
  ```

  Example output:

  ```yaml
  Service: guestbook
    Port: http 80/HTTP targets pod port 3000
  DestinationRule: destination-rule-guestbook for "guestbook"
    Matching subsets: v1,v2
    No Traffic Policy
  Pod is STRICT, clients configured automatically
  ```

## Further Reading

* [Basic TLS/SSL Terminology](https://dzone.com/articles/tlsssl-terminology-and-basics)

* [TLS Handshake Explained](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_7.1.0/com.ibm.mq.doc/sy10660_.htm)

* [Istio Task](https://istio.io/latest/docs/tasks/security/)

* [Istio Concept](https://istio.io/docs/concepts/security/mutual-tls.html)
