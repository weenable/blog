---
title: P2P核心技术： 以太坊P2P连接
date: 2025-04-30T11:16:08+08:00
toc: true
readTime: true
autonumber: true
math: true
categories:
  - web3
showTags: true
hideBackToTop: false
---
### 详细逻辑
1. 建立网络连接
2. ECDH握手
3. 消息加密和签名


### 代码实现
服务端实现
```
package main  
  
import (  
    "bytes"  
    "crypto/aes"    "crypto/cipher"    "crypto/ecdsa"    "crypto/elliptic"    "crypto/rand"    "crypto/sha256"    "errors"    "fmt"    "github.com/ethereum/go-ethereum/crypto"    "github.com/ethereum/go-ethereum/crypto/ecies"    "github.com/ethereum/go-ethereum/p2p"    "github.com/ethereum/go-ethereum/p2p/discover"    "github.com/ethereum/go-ethereum/p2p/enode"    "github.com/ethereum/go-ethereum/p2p/nat"    "github.com/ethereum/go-ethereum/rlp"    "log"    "math/big"    "net")  
  
const (  
    MsgTypeHandshake = 0  
    MsgTypeData      = 1  
)  
  
type SimpleProtocol struct {  
    peers       map[*p2p.Peer]*peerSession  
    privateKey  *ecies.PrivateKey  
    ecdsaPrv    *ecdsa.PrivateKey  
    localPubKey []byte  
}  
  
type peerSession struct {  
    rw           p2p.MsgReadWriter  
    sharedKey    []byte  
    peerPubKey   *ecies.PublicKey  
    peerECDSAPub *ecdsa.PublicKey  
    aesgcm       cipher.AEAD  
}  
  
func (ps *peerSession) decrypt(encrypted []byte) ([]byte, error) {  
    nonceSize := ps.aesgcm.NonceSize()  
    nonce, ct := encrypted[:nonceSize], encrypted[nonceSize:]  
  
    plaintext, err := ps.aesgcm.Open(nil, nonce, ct, nil)  
    if err != nil {  
       return nil, fmt.Errorf("解密失败: %v", err)  
    }  
  
    return plaintext, nil  
}  
  
func (sp *SimpleProtocol) Name() string   { return "SimpleProtocol" }  
func (sp *SimpleProtocol) Version() uint  { return 1 }  
func (sp *SimpleProtocol) Length() uint64 { return 16 }  
  
func (sp *SimpleProtocol) generateSharedSecret(peerPubKey []byte) ([]byte, error) {  
    pubKey, err := crypto.DecompressPubkey(peerPubKey)  
    if err != nil {  
       return nil, err  
    }  
    peerEciesPub := ecies.ImportECDSAPublic(pubKey)  
    localEciesPub := sp.privateKey.PublicKey  
  
    x, _ := localEciesPub.Curve.ScalarMult(peerEciesPub.X, peerEciesPub.Y, sp.privateKey.D.Bytes())  
    if x == nil {  
       return nil, errors.New("failed to generate shared secret")  
    }  
  
    shared := sha256.Sum256(x.Bytes())  
    return shared[:], nil  
}  
func (sp *SimpleProtocol) handshake(peer *p2p.Peer, session *peerSession) error {  
    // 发送公钥  
    if err := p2p.Send(session.rw, MsgTypeHandshake, sp.localPubKey); err != nil {  
       return err  
    }  
  
    // 接受对方公钥  
    msg, err := session.rw.ReadMsg()  
    if err != nil {  
       return err  
    }  
    if msg.Code != MsgTypeHandshake {  
       return errors.New("unexpected message type")  
    }  
  
    var peerPubKey []byte  
    if err = msg.Decode(&peerPubKey); err != nil {  
       return err  
    }  
  
    // 生成共享密钥  
    shared, err := sp.generateSharedSecret(peerPubKey)  
    if err != nil {  
       return err  
    }  
  
    block, err := aes.NewCipher(shared)  
    if err != nil {  
       return err  
    }  
  
    aesgcm, err := cipher.NewGCM(block)  
    if err != nil {  
       return err  
    }  
    session.aesgcm = aesgcm  
  
    pubKey, err := crypto.DecompressPubkey(peerPubKey)  
    if err != nil {  
       return fmt.Errorf("解压公钥失败: %v", err)  
    }  
    session.peerECDSAPub = pubKey  
  
    log.Printf("Handshake completed with %v", peer)  
    return nil  
}  
  
func (sp *SimpleProtocol) handleMessages(peer *p2p.Peer, session *peerSession) error {  
    for {  
       msg, err := session.rw.ReadMsg()  
       if err != nil {  
          log.Printf("Read error: %v", err)  
          return nil  
       }  
       var ciphertext []byte  
       msg.Decode(&ciphertext)  
  
       pt, err := session.decrypt(ciphertext)  
       var signedMsg struct {  
          Data      []byte  
          Signature []byte  
       }  
       s := rlp.NewStream(bytes.NewReader(pt), 0)  
       if err := s.Decode(&signedMsg); err != nil {  
          log.Printf("解析签名消息失败: %v", err)  
          continue  
       }  
       hash := crypto.Keccak256Hash(signedMsg.Data)  
       sig := signedMsg.Signature  
  
       r := new(big.Int).SetBytes(sig[:32])  
       ss := new(big.Int).SetBytes(sig[32:64])  
       valid := ecdsa.Verify(session.peerECDSAPub, hash.Bytes(), r, ss)  
       if !valid {  
          log.Printf("签名验证失败")  
          continue  
       }  
  
       log.Printf("receive msg: %s", string(signedMsg.Data))  
    }  
}  
func (sp *SimpleProtocol) Run(peer *p2p.Peer, rw p2p.MsgReadWriter) error {  
    session := &peerSession{rw: rw}  
    sp.peers[peer] = session  
  
    if err := sp.handshake(peer, session); err != nil {  
       return fmt.Errorf("handshake failed: %v", err)  
    }  
  
    defer delete(sp.peers, peer)  
  
    err := sp.handleMessages(peer, session)  
    if err != nil {  
       return fmt.Errorf("handlemessage failed: %v", err)  
    }  
  
    return nil  
}  
  
func (sp *SimpleProtocol) SendMessage(peer *p2p.Peer, message string) error {  
    session, ok := sp.peers[peer]  
    if !ok {  
       return errors.New("peer not connected")  
    }  
  
    hash := crypto.Keccak256Hash([]byte(message))  
    signature, err := crypto.Sign(hash.Bytes(), sp.ecdsaPrv)  
    if err != nil {  
       return fmt.Errorf("签名失败: %v", err)  
    }  
    signedMsg := struct {  
       Data      []byte  
       Signature []byte  
    }{  
       Data:      []byte(message),  
       Signature: signature,  
    }  
  
    // RLP编码  
    var buffer bytes.Buffer  
    if err := rlp.Encode(&buffer, signedMsg); err != nil {  
       return fmt.Errorf("RLP编码失败: %v", err)  
    }  
    encodedMsg := buffer.Bytes()  
  
    // 加密  
    nonce := make([]byte, session.aesgcm.NonceSize())  
    if _, err := rand.Read(nonce); err != nil {  
       return fmt.Errorf("生成nonce失败: %v", err)  
    }  
    ciphertext := session.aesgcm.Seal(nil, nonce, encodedMsg, nil)  
  
    // 合并nonce和密文  
    encryptedMsg := append(nonce, ciphertext...)  
  
    // 发送加密消息  
    return p2p.Send(session.rw, MsgTypeData, encryptedMsg)  
}  
  
func NewSimpleProtocol(prv *ecdsa.PrivateKey) *SimpleProtocol {  
    eciesPrv := ecies.ImportECDSA(prv)  
    pubKey := elliptic.MarshalCompressed(prv.Curve, prv.PublicKey.X, prv.PublicKey.Y)  
    return &SimpleProtocol{  
       peers:       make(map[*p2p.Peer]*peerSession),  
       privateKey:  eciesPrv,  
       localPubKey: pubKey,  
    }  
}  
  
func main() {  
    privKey, _ := crypto.GenerateKey()  
    proto := NewSimpleProtocol(privKey)  
  
    config := p2p.Config{  
       PrivateKey: privKey,  
       Name:       "SimpleP2PNode",  
       ListenAddr: ":30303", // 固定端口  
       Protocols: []p2p.Protocol{{  
          Name:    proto.Name(),  
          Version: proto.Version(),  
          Length:  proto.Length(),  
          Run:     proto.Run,  
       }},  
       NAT:             nat.Any(),  
       MaxPeers:        10,  
       EnableMsgEvents: true,  
    }  
  
    srv := &p2p.Server{Config: config}  
    if err := srv.Start(); err != nil {  
       log.Fatalf("Failed to start server: %v", err)  
    }  
    defer srv.Stop()  
  
    log.Printf("Server enode: %s", srv.Self().URLv4())  
  
    // 初始化发现协议  
    tcpAddr, _ := net.ResolveTCPAddr("tcp", srv.ListenAddr)  
    udpPort := tcpAddr.Port + 1  
    udpAddr := fmt.Sprintf(":%d", udpPort)  
    udpConn, err := net.ListenPacket("udp", udpAddr)  
    if err != nil {  
       log.Fatalf("Failed to listen UDP: %v", err)  
    }  
  
    discoverDB, _ := enode.OpenDB("")  
    localNode := enode.NewLocalNode(discoverDB, privKey)  
    localNode.SetFallbackIP(net.IP{127, 0, 0, 1})  
    localNode.SetFallbackUDP(udpPort)  
  
    discv5, err := discover.ListenV5(udpConn.(*net.UDPConn), localNode, discover.Config{  
       PrivateKey: privKey,  
    })  
    if err != nil {  
       log.Fatalf("Failed to start discovery: %v", err)  
    }  
    defer discv5.Close()  
  
    select {}  
}
```

客户端实现
```
package main  
  
import (  
    "bufio"  
    "bytes"    "crypto/aes"    "crypto/cipher"    "crypto/ecdsa"    "crypto/elliptic"    "crypto/rand"    "crypto/sha256"    "encoding/gob"    "errors"    "flag"    "fmt"    "github.com/ethereum/go-ethereum/rlp"    "log"    "net"    "os"  
    "github.com/ethereum/go-ethereum/crypto"    "github.com/ethereum/go-ethereum/crypto/ecies"    "github.com/ethereum/go-ethereum/p2p"    "github.com/ethereum/go-ethereum/p2p/enode"    "github.com/ethereum/go-ethereum/p2p/nat")  
  
const (  
    MsgTypeHandshake = 0  
    MsgTypeData      = 1  
)  
  
type SecureClient struct {  
    peers       map[*p2p.Peer]*peerSession  
    privateKey  *ecies.PrivateKey  
    ecdsaPrv    *ecdsa.PrivateKey  
    localPubKey []byte  
    targetNode  *enode.Node  
}  
  
type peerSession struct {  
    rw         p2p.MsgReadWriter  
    aesgcm     cipher.AEAD  
    peerPubKey *ecies.PublicKey  
}  
  
var targetEnode = "enode://51631475f5e373db90aede61c14046a54efc2867e6850f036f5a3645e904785a2c5445e706eb66bc1380fe4eceb645e1e9effe59f6779a22ab6b7a978b6ad865@127.0.0.1:30303"  
  
func main() {  
    flag.Parse()  
    gob.Register(&net.TCPAddr{})  
    gob.Register(&net.UDPAddr{})  
  
    // 解析目标节点  
    node, err := enode.ParseV4(targetEnode)  
    if err != nil {  
       log.Fatalf("解析enode失败: %v", err)  
    }  
  
    // 生成客户端密钥  
    privKey, _ := crypto.GenerateKey()  
    client := NewSecureClient(privKey, node)  
  
    config := p2p.Config{  
       PrivateKey: privKey,  
       Name:       "SecureClient",  
       ListenAddr: ":0",  
       Protocols: []p2p.Protocol{{  
          Name:    client.Name(),  
          Version: client.Version(),  
          Length:  client.Length(),  
          Run:     client.Run,  
       }},  
       NAT:         nat.Any(),  
       MaxPeers:    1,  
       StaticNodes: []*enode.Node{node},  
    }  
  
    // 启动客户端  
    srv := &p2p.Server{Config: config}  
    if err := srv.Start(); err != nil {  
       log.Fatal(err)  
    }  
    defer srv.Stop()  
  
    log.Printf("客户端已启动，enode: %s", srv.Self().URLv4())  
  
    // 命令行交互  
    go client.startCLI(srv)  
  
    select {}  
}  
  
func NewSecureClient(prv *ecdsa.PrivateKey, target *enode.Node) *SecureClient {  
    eciesPrv := ecies.ImportECDSA(prv)  
    pubKey := elliptic.MarshalCompressed(prv.Curve, prv.PublicKey.X, prv.PublicKey.Y)  
    return &SecureClient{  
       peers:       make(map[*p2p.Peer]*peerSession),  
       privateKey:  eciesPrv,  
       localPubKey: pubKey,  
       targetNode:  target,  
       ecdsaPrv:    prv,  
    }  
}  
  
func (c *SecureClient) Name() string   { return "SimpleProtocol" }  
func (c *SecureClient) Version() uint  { return 1 }  
func (c *SecureClient) Length() uint64 { return 16 }  
  
func (c *SecureClient) Run(peer *p2p.Peer, rw p2p.MsgReadWriter) error {  
    session := &peerSession{rw: rw}  
    c.peers[peer] = session  
  
    // 执行握手协议  
    if err := c.performHandshake(peer, session); err != nil {  
       return fmt.Errorf("握手失败: %v", err)  
    }  
  
    // 启动消息接收循环  
    c.handleMessages(peer, session)  
  
    delete(c.peers, peer)  
    return nil  
}  
  
func (c *SecureClient) performHandshake(peer *p2p.Peer, session *peerSession) error {  
    // 发送公钥  
    if err := p2p.Send(session.rw, MsgTypeHandshake, c.localPubKey); err != nil {  
       return err  
    }  
  
    // 接收服务端公钥  
    msg, err := session.rw.ReadMsg()  
    if err != nil {  
       return err  
    }  
    if msg.Code != MsgTypeHandshake {  
       return errors.New("非预期的消息类型")  
    }  
  
    var serverPubKey []byte  
    if err := msg.Decode(&serverPubKey); err != nil {  
       return err  
    }  
  
    // 生成共享密钥  
    shared, err := c.generateSharedSecret(serverPubKey)  
    if err != nil {  
       return err  
    }  
  
    // 初始化AES-GCM  
    block, err := aes.NewCipher(shared)  
    if err != nil {  
       return err  
    }  
  
    aesgcm, err := cipher.NewGCM(block)  
    if err != nil {  
       return err  
    }  
    session.aesgcm = aesgcm  
  
    log.Printf("与服务端 %s 握手成功", peer)  
    return nil  
}  
  
func (c *SecureClient) generateSharedSecret(peerPubKey []byte) ([]byte, error) {  
    pub, err := crypto.DecompressPubkey(peerPubKey)  
    if err != nil {  
       return nil, err  
    }  
    eciesPub := ecies.ImportECDSAPublic(pub)  
  
    // 椭圆曲线点乘法  
    x, _ := eciesPub.Curve.ScalarMult(eciesPub.X, eciesPub.Y, c.privateKey.D.Bytes())  
    if x == nil {  
       return nil, errors.New("共享密钥生成失败")  
    }  
  
    // 使用SHA-256派生密钥  
    shared := sha256.Sum256(x.Bytes())  
    return shared[:], nil  
}  
  
func (c *SecureClient) handleMessages(peer *p2p.Peer, session *peerSession) {  
    for {  
       msg, err := session.rw.ReadMsg()  
       if err != nil {  
          log.Printf("读取错误: %v", err)  
          return  
       }  
  
       switch msg.Code {  
       case MsgTypeData:  
          var encrypted []byte  
          if err := msg.Decode(&encrypted); err != nil {  
             log.Printf("解码错误: %v", err)  
             continue  
          }  
          log.Printf("[来自服务端] %s", string(encrypted))  
  
       default:  
          log.Printf("未知消息类型: %d", msg.Code)  
       }  
    }  
}  
  
func (sc *SecureClient) SendMessage(peer *p2p.Peer, message string) error {  
    session, ok := sc.peers[peer]  
    if !ok {  
       return errors.New("peer not connected")  
    }  
  
    hash := crypto.Keccak256Hash([]byte(message))  
    signature, err := crypto.Sign(hash.Bytes(), sc.ecdsaPrv)  
    if err != nil {  
       return fmt.Errorf("签名失败: %v", err)  
    }  
    signedMsg := struct {  
       Data      []byte  
       Signature []byte  
    }{  
       Data:      []byte(message),  
       Signature: signature,  
    }  
  
    // RLP编码  
    var buffer bytes.Buffer  
    if err := rlp.Encode(&buffer, signedMsg); err != nil {  
       return fmt.Errorf("RLP编码失败: %v", err)  
    }  
    encodedMsg := buffer.Bytes()  
  
    // 加密  
    nonce := make([]byte, session.aesgcm.NonceSize())  
    if _, err := rand.Read(nonce); err != nil {  
       return fmt.Errorf("生成nonce失败: %v", err)  
    }  
    ciphertext := session.aesgcm.Seal(nil, nonce, encodedMsg, nil)  
  
    // 合并nonce和密文  
    encryptedMsg := append(nonce, ciphertext...)  
  
    // 发送加密消息  
    return p2p.Send(session.rw, MsgTypeData, encryptedMsg)  
}  
  
func (c *SecureClient) startCLI(srv *p2p.Server) {  
    reader := bufio.NewReader(os.Stdin)  
    for {  
       fmt.Print("输入消息 > ")  
       text, _ := reader.ReadString('\n')  
  
       if len(srv.Peers()) == 0 {  
          log.Println("尚未连接到任何节点")  
          continue  
       }  
  
       peer := srv.Peers()[0]  
       err := c.SendMessage(peer, text)  
       if err != nil {  
          log.Printf("发送失败：%v", err)  
       }  
    }  
}
```