---
title: step-ca证书管理工具的使用
date: 2023-03-22T19:06:00+08:00
lastmod: 2023-03-22T19:06:00+08:00
author: sasaba
cover: /img/使用cfssl生成tls证书.jpg
images:
- /img/使用cfssl生成tls证书.jpg
categories:
  - 工具
tags:
  - SSL
  - ACME
draft: true
---

step-ca的使用教程。

<!--more-->

## 说明
[step-ca](https://github.com/smallstep/certificates)是安全、自动化证书管理的在线证书颁发机构。

分为两个子应用，step-ca作为daemon程序运行，提供证书签发、acme等功能。

step-cli是一个cli工具用来和step-ca交互。

## 功能
step-ca的功能相对强大，本文主要介绍它的搭建和证书签发以及acme的示例。

详细可以看[文档](https://smallstep.com/docs/step-ca)

## 安装
二进制文件安装

[step-ca](https://github.com/smallstep/certificates/releases)

[step-cli](https://github.com/smallstep/cli/releases)

对应操作系统下载二进制文件，并加入/usr/bin或windows的PATH

### 以linux为例演示如何配置

1. 创建一个step用户

```shell
# 添加一个step用户
sudo useradd --system --home /home/step --shell /bin/bash step
sudo mkdir -p /home/step && sudo chown -R step:step /home/step

# 授权可以使用特权端口
sudo setcap CAP_NET_BIND_SERVICE=+eip $(which step-ca)
```

2. 切换用户并初始化ca证书
```shell
step@ubuntu2004:~$ step ca init

✔ Deployment Type: Standalone
What would you like to name your new PKI?
✔ (e.g. Smallstep): demo.com
What DNS names or IP addresses will clients use to reach your CA?
✔ (e.g. ca.example.com[,10.1.2.3,etc.]): ca.demo.com
What IP and port will your new CA bind to? (:443 will bind to 0.0.0.0:443)
✔ (e.g. :443 or 127.0.0.1:443): :443
What would you like to name the CA's first provisioner?
✔ (e.g. you@smallstep.com): admin@demo.com
Choose a password for your CA keys and first provisioner.
✔ [leave empty and we'll generate one]: 
✔ Password: s(SB_m<%3BN3H`<MzP\Dw&a,Ycg04_<c

Generating root certificate... done!
Generating intermediate certificate... done!

✔ Root certificate: /home/step/.step/certs/root_ca.crt
✔ Root private key: /home/step/.step/secrets/root_ca_key
✔ Root fingerprint: 08f9a9b89c9238cb1bff7bcf4086ef3f3cc58680f39f7c8dec8449270f0997b9
✔ Intermediate certificate: /home/step/.step/certs/intermediate_ca.crt
✔ Intermediate private key: /home/step/.step/secrets/intermediate_ca_key
✔ Database folder: /home/step/.step/db
✔ Default configuration: /home/step/.step/config/defaults.json
✔ Certificate Authority configuration: /home/step/.step/config/ca.json

Your PKI is ready to go. To generate certificates for individual services see 'step help ca'.

FEEDBACK 😍 🍻
  The step utility is not instrumented for usage statistics. It does not phone
  home. But your feedback is extremely valuable. Any information you can provide
  regarding how you’re using `step` helps. Please send us a sentence or two,
  good or bad at feedback@smallstep.com or join GitHub Discussions
  https://github.com/smallstep/certificates/discussions and our Discord 
  https://u.step.sm/discord.
```
主要填写下PKI name和机构的地址(我这里写的是ca.demo.com)，同时会生成一个密码和指纹信息。密码需要记录。

3. 使用systemd注册为服务(注意使用特权用户)
```shell
sudo touch /etc/systemd/system/step-ca.service
```

> /etc/systemd/system/step-ca.service

这里要注意/home/step/.step要替换成实际的地址

/home/step/.step/password.txt需要将刚才生成的密码输入进去 `echo ${password} > /home/step/.step/password.txt`
```text
[Unit]
Description=step-ca service
Documentation=https://smallstep.com/docs/step-ca
Documentation=https://smallstep.com/docs/step-ca/certificate-authority-server-production
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=30
StartLimitBurst=3
ConditionFileNotEmpty=/home/step/.step/config/ca.json
ConditionFileNotEmpty=/home/step/.step/password.txt

[Service]
Type=simple
User=step
Group=step
Environment=STEPPATH=/home/step/.step
WorkingDirectory=/home/step/.step
ExecStart=/usr/bin/step-ca config/ca.json --password-file password.txt
ExecReload=/bin/kill --signal HUP $MAINPID
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=30
StartLimitBurst=3

; Process capabilities & privileges
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
SecureBits=keep-caps
NoNewPrivileges=yes

; Sandboxing
ProtectSystem=full
ProtectHome=false
RestrictNamespaces=true
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
PrivateTmp=true
PrivateDevices=true
ProtectClock=true
ProtectControlGroups=true
ProtectKernelTunables=true
ProtectKernelLogs=true
ProtectKernelModules=true
LockPersonality=true
RestrictSUIDSGID=true
RemoveIPC=true
RestrictRealtime=true
SystemCallFilter=@system-service
SystemCallArchitectures=native
MemoryDenyWriteExecute=true
ReadWriteDirectories=/home/step/.step/db

[Install]
WantedBy=multi-user.target
```

```shell
# 启动
systemctl enable --now step-ca

# 查看状态
systemctl status step-ca
journalctl --follow --unit=step-ca
```

## 使用step-ca和step生成证书
### 下载证书(远端)

```shell
# 在安装step-ca的机器使用
step certificate fingerprint $(step path)/certs/root_ca.crt 
# 指纹：db95a8117e5cdfa03a52c11978c70fb4399ef5587c16ef715e9bdae0a0e8df73

step ca bootstrap --ca-url https://ca.demo.com --fingerprint db95a8117e5cdfa03a52c11978c70fb4399ef5587c16ef715e9bdae0a0e8df73
```

### 签发证书
```shell
# 注意远程操作需要输入密码 而在step-ca机器上则不要
# 基本用法 默认只有一天的有效期
step ca certificate api.demo.com srv.crt srv.key

# 可以通过inspect查看
step certificate inspect srv.crt --short

# 这个证书5分钟后过期
step ca certificate api.demo.com srv.crt srv.key --not-after=5m

# 这个证书5分钟后生效，240h后过期
step ca certificate api.demo.com srv.crt srv.key --not-before=5m --not-after=240h

# 吊销证书
step ca revoke --cert svc.crt --key svc.key
```

### 信任ca
```shell
step certificate install $(step path)/certs/root_ca.crt
```

### 测试证书
```go
package main

import (
	"io"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		io.WriteString(w, "hello, world!\n")
	})
	if e := http.ListenAndServeTLS(":443", "svc.crt", "svc.key", nil); e != nil {
		log.Fatal("ListenAndServe: ", e)
	}
}
```
## acme

### 安装came的provider(需要在安装step-ca的机器上)
```shell
step ca provisioner add acme --type ACME

sudo systemctl restart step-ca
```

### 测试
golang的例子
```go
package main

import (
	"crypto"
	"crypto/ecdsa"
	"crypto/elliptic"
	"crypto/rand"
	"crypto/tls"
	"crypto/x509"
	"errors"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"sync"
	"time"

	"github.com/go-acme/lego/certcrypto"
	"github.com/go-acme/lego/certificate"
	"github.com/go-acme/lego/challenge/http01"
	"github.com/go-acme/lego/lego"
	"github.com/go-acme/lego/registration"
	"golang.org/x/net/http2"
)

const (
	// The domain name for which we'll be getting a certificate
	domain = "test.demo.com"

	// The email address to use during ACME registration
	acmeEmail = "test@demo.com"

	// The ACME directory URL for your ACME server
	acmeDirectoryURL = "https://ca.demo.com/acme/acme/directory"

	// The root certificate for the CA that issued the ACME server's
	// certificate.
	rootCertificate = "C:\\Users\\sasaba\\.step\\certs\\root_ca.crt"

	// How frequently we should check whether our cert needs renewal
	tickFrequency = 15 * time.Second

	// The listen address for our HTTPS server
	listenAddr = ":5443"
)

// loadRootCertPool builds a trust store (cert pool) containing our CA's root
// certificate.
func loadRootCertPool(rootCert string) (*x509.CertPool, error) {
	root, err := ioutil.ReadFile(rootCert)
	if err != nil {
		return nil, err
	}

	pool := x509.NewCertPool()
	if ok := pool.AppendCertsFromPEM(root); !ok {
		return nil, errors.New("Missing or invalid root certificate")
	}

	return pool, nil
}

// getHTTPSClient gets an HTTPS client configured to trust our CA's root
// certificate.
func getHTTPSClient(rootCert string) (*http.Client, error) {
	pool, err := loadRootCertPool(rootCert)
	if err != nil {
		return nil, err
	}

	tr := &http.Transport{
		TLSClientConfig: &tls.Config{
			MinVersion:               tls.VersionTLS12,
			PreferServerCipherSuites: true,
			RootCAs:                  pool,
		},
	}
	if err := http2.ConfigureTransport(tr); err != nil {
		return nil, errors.New("Error configuring transport")
	}
	return &http.Client{
		Transport: tr,
	}, nil
}

// LegoUser implements registration.User, required by lego.
type LegoUser struct {
	email        string
	registration *registration.Resource
	key          crypto.PrivateKey
}

func (l *LegoUser) GetEmail() string {
	return l.email
}

func (l *LegoUser) GetRegistration() *registration.Resource {
	return l.registration
}

func (l *LegoUser) GetPrivateKey() crypto.PrivateKey {
	return l.key
}

// Uses techniques from https://diogomonica.com/2017/01/11/hitless-tls-certificate-rotation-in-go/
// to automatically rotate certificates when they're renewed.

// ACMECertManager manages ACME certificate renewals and makes it easy to use
// certificates with the tls package.`
type ACMECertManager struct {
	sync.RWMutex
	acmeClient  *lego.Client
	certificate *tls.Certificate
	domains     []string
	leaf        *x509.Certificate
	resource    *certificate.Resource
}

// NewACMECertManager configures an ACME client, creates & registers a new ACME
// user. After creating a client you must call ObtainCertificate and
// RenewCertificate yourself.
func NewACMECertManager(domains []string, email, rootCert, caDirURL string) (*ACMECertManager, error) {
	// Create a new ACME user with a new key.
	key, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	if err != nil {
		return nil, err
	}
	user := &LegoUser{
		email: email,
		key:   key,
	}

	// Get an HTTPS client configured to trust our root certificate.
	httpClient, err := getHTTPSClient(rootCert)
	if err != nil {
		return nil, err
	}

	// Create a configuration using our HTTPS client, ACME server, user details.
	config := &lego.Config{
		CADirURL:   caDirURL,
		User:       user,
		HTTPClient: httpClient,
		Certificate: lego.CertificateConfig{
			KeyType: certcrypto.RSA2048,
			Timeout: 30 * time.Second,
		},
	}

	// Create an ACME client and configure use of `http-01` challenge
	acmeClient, err := lego.NewClient(config)
	if err != nil {
		return nil, err
	}
	err = acmeClient.Challenge.SetHTTP01Provider(http01.NewProviderServer("", "80"))
	if err != nil {
		log.Fatal(err)
	}

	// Register our ACME user
	registration, err := acmeClient.Registration.Register(registration.RegisterOptions{TermsOfServiceAgreed: true})
	if err != nil {
		return nil, err
	}
	user.registration = registration

	return &ACMECertManager{
		acmeClient: acmeClient,
		domains:    domains,
	}, nil
}

// ObtainCertificate gets a new certificate using ACME. Not thread safe.
func (a *ACMECertManager) ObtainCertificate() error {
	request := certificate.ObtainRequest{
		Domains: a.domains,
		Bundle:  true,
	}

	resource, err := a.acmeClient.Certificate.Obtain(request)
	if err != nil {
		return err
	}

	return a.switchCertificate(resource)
}

// RenewCertificate renews an existing certificate using ACME. Not thread safe.
func (a *ACMECertManager) RenewCertificate() error {
	resource, err := a.acmeClient.Certificate.Renew(*a.resource, true, false)
	if err != nil {
		return err
	}

	return a.switchCertificate(resource)
}

// GetCertificate locks around returning a tls.Certificate; use as tls.Config.GetCertificate.
func (a *ACMECertManager) GetCertificate(hello *tls.ClientHelloInfo) (*tls.Certificate, error) {
	a.RLock()
	defer a.RUnlock()
	return a.certificate, nil
}

// GetLeaf returns the currently valid leaf x509.Certificate
func (a *ACMECertManager) GetLeaf() x509.Certificate {
	a.RLock()
	defer a.RUnlock()
	return *a.leaf
}

// NextRenewal returns when the certificate will be 2/3 of the way to expiration.
func (a *ACMECertManager) NextRenewal() time.Time {
	leaf := a.GetLeaf()
	lifetime := leaf.NotAfter.Sub(leaf.NotBefore).Seconds()
	return leaf.NotBefore.Add(time.Duration(lifetime*2/3) * time.Second)
}

// NeedsRenewal returns true if the certificate's age is more than 2/3 it's
// lifetime.
func (a *ACMECertManager) NeedsRenewal() bool {
	return time.Now().After(a.NextRenewal())
}

func (a *ACMECertManager) switchCertificate(newResource *certificate.Resource) error {
	// The certificate.Resource represents our certificate as a PEM-encoded
	// bundle of bytes. Let's process it. First create a tls.Certificate
	// for use with the tls package.
	crt, err := tls.X509KeyPair(newResource.Certificate, newResource.PrivateKey)
	if err != nil {
		return err
	}

	// Now create an x509.Certificate so we can figure out when the cert
	// expires. Note that the first certificate in the bundle is the leaf.
	// Go ahead and set crt.Leaf as an optimization.
	leaf, err := x509.ParseCertificate(crt.Certificate[0])
	if err != nil {
		return err
	}
	crt.Leaf = leaf

	a.Lock()
	defer a.Unlock()
	a.resource = newResource
	a.certificate = &crt
	a.leaf = leaf

	return nil
}

func main() {
	// Let's create a little web server that responds to TLS or mutually
	// authenticated TLS connections. If we get a mutual TLS connection
	// we'll respond with a friendly greeting using the client's name.
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		if r.TLS == nil || len(r.TLS.PeerCertificates) == 0 {
			fmt.Fprintf(w, "Hello, TLS!\n")
		} else {
			name := r.TLS.PeerCertificates[0].Subject.CommonName
			fmt.Fprintf(w, "Hello, %s!\n", name)
		}
	})

	// Create a trust pool with our CA's root certificate; used to validate
	// client certificates.
	roots, err := loadRootCertPool(rootCertificate)
	if err != nil {
		log.Fatal(err)
	}

	// Configure our ACME cert manager and get a certificate using ACME!
	acm, err := NewACMECertManager([]string{domain}, acmeEmail, rootCertificate, acmeDirectoryURL)
	if err != nil {
		log.Fatal("Error creating ACMECertManager", err)
	}
	err = acm.ObtainCertificate()
	if err != nil {
		log.Fatal("Error loading certificate and key", err)
	}

	// Create a TLS configuration for our HTTPS server. The fancy bits here
	// are commented.
	cfg := &tls.Config{
		// This makes client authentication optional. Switch to
		// tls.RequireAndVerifyClientCert to require authentication.
		ClientAuth: tls.VerifyClientCertIfGiven,

		// We'll be trusting client certificates issued by our CA.
		ClientCAs: roots,

		MinVersion:               tls.VersionTLS12,
		CurvePreferences:         []tls.CurveID{tls.CurveP521, tls.CurveP384, tls.CurveP256},
		PreferServerCipherSuites: true,
		CipherSuites: []uint16{
			tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,
			tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
			tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
		},

		// Dynamically load certificate from ACMECertManager with every
		// connection, so renewals work.
		GetCertificate: acm.GetCertificate,
	}

	// Make a server!
	srv := &http.Server{
		Addr:      listenAddr,
		Handler:   mux,
		TLSConfig: cfg,
	}

	// Schedule periodic certificate renewal (do ACME again periodically).
	// We'll tick every timeFrequency but only renew if the certificate
	// is approaching expiration. That'll give us some resilience to CA
	// downtime.
	done := make(chan struct{})
	go func() {
		ticker := time.NewTicker(tickFrequency)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				if acm.NeedsRenewal() {
					fmt.Println("Renewing certificate")
					err := acm.RenewCertificate()
					if err != nil {
						log.Println("Error loading certificate and key", err)
					} else {
						leaf := acm.GetLeaf()
						fmt.Printf("Renewed certificate: %s [%s - %s]\n", leaf.Subject, leaf.NotBefore, leaf.NotAfter)
						fmt.Printf("Next renewal at %s (%s)\n", acm.NextRenewal(), acm.NextRenewal().Sub(time.Now()))
					}
				} else {
					fmt.Printf("Waiting to renew at %s (%s)\n", acm.NextRenewal(), acm.NextRenewal().Sub(time.Now()))
				}
			case <-done:
				return
			}
		}
	}()
	defer close(done)

	log.Printf("Listening on %s\n", listenAddr)

	// Start serving HTTPS!
	err = srv.ListenAndServeTLS("", "")
	if err != nil {
		log.Fatal("ListenAndServerTLS: ", err)
	}
}
```
> 注意test.demo.com必须要能被step-ca的服务器通过hosts或者dns的方式访问

### 文档
更多acme客户端的教程可以看这个[文档](https://smallstep.com/docs/tutorials/acme-protocol-acme-clients)