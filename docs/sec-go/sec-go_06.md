# 第六章：密码学

加密是一种在第三方可以查看通信时保护通信的实践。有双向对称和非对称加密方法，以及单向哈希算法。

加密是现代互联网的关键部分。有了[LetsEncrypt.com](http://www.LetsEncrypt.com)等服务，每个人都可以获得受信任的 SSL 证书。我们的整个基础设施都依赖于加密来保护所有机密数据。正确加密和正确哈希数据非常重要，而且很容易配置错误的服务，使其容易受到攻击或暴露。

本章涵盖以下示例和用例：

+   对称和非对称加密

+   签名和验证消息

+   哈希处理

+   安全存储密码

+   生成安全的随机数

+   创建和使用 TLS/SSL 证书

# 哈希处理

哈希是将可变长度消息转换为唯一的固定长度的字母数字字符串。有各种可用的哈希算法，如 MD5 和 SHA1。哈希是单向且不可逆的，不像对称加密函数（如 AES），如果您有密钥，可以恢复原始消息。由于哈希无法被反转，大多数哈希都会被暴力破解。破解者将构建功耗巨大的装置，配备多个 GPU，以对每个可能的字符组合进行哈希，直到找到与之匹配的哈希。他们还会生成彩虹表或包含所有已生成哈希输出的文件，以进行快速查找。

因此，对哈希进行加盐很重要。加盐是向用户提供的密码末尾添加随机字符串的过程，以增加更多的随机性或熵。考虑一个存储用户登录信息和哈希密码以进行身份验证的应用程序。如果两个用户使用相同的密码，则它们的哈希输出将相同。没有盐，破解者可能会找到多个使用相同密码的人，并且只需要破解一次哈希。通过为每个用户的密码添加唯一的盐，您确保每个用户都具有唯一的哈希值。加盐减少了彩虹表的有效性，因为即使他们知道与每个哈希相关的盐，他们也必须为每个盐生成一个彩虹表，这是耗时的。

哈希通常用于验证密码。另一个常见用途是用于文件完整性。大型下载通常附带文件的 MD5 或 SHA1 哈希。下载后，您可以对文件进行哈希处理，以确保其与预期值匹配。如果不匹配，则下载文件已被修改。哈希还用作记录妥协指标或 IOC 的一种方式。已知恶意或危险的文件会被哈希处理，并且该哈希将存储在目录中。这些通常是公开共享的，以便人们可以检查可疑文件是否存在已知风险。与整个文件相比，存储和比较哈希要高效得多。

# 对小文件进行哈希处理

如果文件小到可以包含在内存中，`ReadFile()`方法可以快速工作。它将整个文件加载到内存中，然后对数据进行摘要。将使用多种不同的哈希算法进行计算：

```go
package main

import (
   "crypto/md5"
   "crypto/sha1"
   "crypto/sha256"
   "crypto/sha512"
   "fmt"
   "io/ioutil"
   "log"
   "os"
)

func printUsage() {
   fmt.Println("Usage: " + os.Args[0] + " <filepath>")
   fmt.Println("Example: " + os.Args[0] + " document.txt")
}

func checkArgs() string {
   if len(os.Args) < 2 {
      printUsage()
      os.Exit(1)
   }
   return os.Args[1]
}

func main() {
   filename := checkArgs()

   // Get bytes from file
   data, err := ioutil.ReadFile(filename)
   if err != nil {
      log.Fatal(err)
   }

   // Hash the file and output results
   fmt.Printf("Md5: %x\n\n", md5.Sum(data))
   fmt.Printf("Sha1: %x\n\n", sha1.Sum(data))
   fmt.Printf("Sha256: %x\n\n", sha256.Sum256(data))
   fmt.Printf("Sha512: %x\n\n", sha512.Sum512(data))
}
```

# 对大文件进行哈希处理

在前面的哈希示例中，要进行哈希处理的整个文件在哈希之前被加载到内存中。当文件达到一定大小时，这是不切实际甚至不可能的。物理内存限制将起作用。因为哈希是作为块密码实现的，它将一次操作一个块，而无需一次性将整个文件加载到内存中：

```go
package main

import (
   "crypto/md5"
   "fmt"
   "io"
   "log"
   "os"
)

func printUsage() {
   fmt.Println("Usage: " + os.Args[0] + " <filename>")
   fmt.Println("Example: " + os.Args[0] + " diskimage.iso")
}

func checkArgs() string {
   if len(os.Args) < 2 {
      printUsage()
      os.Exit(1)
   }
   return os.Args[1]
}

func main() {
   filename := checkArgs()

   // Open file for reading
   file, err := os.Open(filename)
   if err != nil {
      log.Fatal(err)
   }
   defer file.Close()

   // Create new hasher, which is a writer interface
   hasher := md5.New()

   // Default buffer size for copying is 32*1024 or 32kb per copy
   // Use io.CopyBuffer() if you want to specify the buffer to use
   // It will write 32kb at a time to the digest/hash until EOF
   // The hasher implements a Write() function making it satisfy
   // the writer interface. The Write() function performs the digest
   // at the time the data is copied/written to it. It digests
   // and processes the hash one chunk at a time as it is received.
   _, err = io.Copy(hasher, file)
   if err != nil {
      log.Fatal(err)
   }

   // Now get the final sum or checksum.
   // We pass nil to the Sum() function because
   // we already copied the bytes via the Copy to the
   // writer interface and don't need to pass any new bytes
   checksum := hasher.Sum(nil)

   fmt.Printf("Md5 checksum: %x\n", checksum)
}
```

# 安全存储密码

现在我们知道如何哈希，我们可以谈论安全地存储密码。哈希是保护密码的重要因素。其他重要因素是加盐，使用密码学强哈希函数，以及可选使用**基于哈希的消息认证码**（HMAC），这些都在哈希算法中添加了一个额外的秘密密钥。

HMAC 是一个使用秘钥的附加层；因此，即使攻击者获得了带有盐的哈希密码数据库，没有秘密密钥，他们仍然会很难破解它们。秘密密钥应该存储在一个单独的位置，比如环境变量，而不是与哈希密码和盐一起存储在数据库中。

这个示例应用程序的用途有限。将其用作您自己应用程序的参考。

```go
package main

import (
   "crypto/hmac"
   "crypto/rand"
   "crypto/sha256"
   "encoding/base64"
   "encoding/hex"
   "fmt"
   "io"
   "os"
)

func printUsage() {
   fmt.Println("Usage: " + os.Args[0] + " <password>")
   fmt.Println("Example: " + os.Args[0] + " Password1!")
}

func checkArgs() string {
   if len(os.Args) < 2 {
      printUsage()
      os.Exit(1)
   }
   return os.Args[1]
}

// secretKey should be unique, protected, private,
// and not hard-coded like this. Store in environment var
// or in a secure configuration file.
// This is an arbitrary key that should only be used 
// for example purposes.
var secretKey = "neictr98y85klfgneghre"

// Create a salt string with 32 bytes of crypto/rand data
func generateSalt() string {
   randomBytes := make([]byte, 32)
   _, err := rand.Read(randomBytes)
   if err != nil {
      return ""
   }
   return base64.URLEncoding.EncodeToString(randomBytes)
}

// Hash a password with the salt
func hashPassword(plainText string, salt string) string {
   hash := hmac.New(sha256.New, []byte(secretKey))
   io.WriteString(hash, plainText+salt)
   hashedValue := hash.Sum(nil)
   return hex.EncodeToString(hashedValue)
}

func main() {
   // Get the password from command line argument
   password := checkArgs()
   salt := generateSalt()
   hashedPassword := hashPassword(password, salt)
   fmt.Println("Password: " + password)
   fmt.Println("Salt: " + salt)
   fmt.Println("Hashed password: " + hashedPassword)
}
```

# 加密

加密与哈希不同，因为它是可逆的，原始消息可以被恢复。有对称加密方法使用密码或共享密钥进行加密和解密。还有非对称加密算法使用公钥和私钥对。AES 是对称加密的一个例子，用于加密 ZIP 文件、PDF 文件或整个文件系统。RSA 是非对称加密的一个例子，用于 SSL、SSH 密钥和 PGP。

# 密码学安全伪随机数生成器（CSPRNG）

`math`和`rand`包提供的随机性不如`crypto/rand`包。不要将`math/rand`用于加密应用。

在[`golang.org/pkg/crypto/rand/`](https://golang.org/pkg/crypto/rand/)上了解更多关于 Go 的`crypto/rand`包的信息。

以下示例将演示如何生成随机字节、随机整数或任何其他有符号或无符号类型的整数：

```go
package main

import (
   "crypto/rand"
   "encoding/binary"
   "fmt"
   "log"
   "math"
   "math/big"
)

func main() {
   // Generate a random int
   limit := int64(math.MaxInt64) // Highest random number allowed
   randInt, err := rand.Int(rand.Reader, big.NewInt(limit))
   if err != nil {
      log.Fatal(err)
   }
   fmt.Println("Random int value: ", randInt)

   // Alternatively, you could generate the random bytes
   // and turn them into the specific data type needed.
   // binary.Read() will only read enough bytes to fill the data type
   var number uint32
   err = binary.Read(rand.Reader, binary.BigEndian, &number)
   if err != nil {
      log.Fatal(err)
   }
   fmt.Println("Random uint32 value: ", number)

   // Or just generate a random byte slice
   numBytes := 4
   randomBytes := make([]byte, numBytes)
   rand.Read(randomBytes)
   fmt.Println("Random byte values: ", randomBytes)
}
```

# 对称加密

对称加密是指使用相同的密钥或密码来加密和解密数据。高级加密标准，也称为 AES 或 Rijndael，是 NIST 在 2001 年制定的对称加密算法标准。

数据加密标准（DES）是另一种对称加密算法，比 AES 更老且不太安全。除非有特定要求或规范，否则不应该使用 DES 而不是 AES。Go 标准库包括 AES 和 DES 包。

# AES

该程序将使用一个 32 字节（256 位）的密码来加密和解密文件。

在生成密钥、加密或解密时，输出通常发送到`STDOUT`或终端。您可以使用`>`运算符将输出轻松重定向到文件或另一个程序。请参考用法模式以获取示例。如果需要将密钥或加密数据存储为 ASCII 编码的字符串，请使用 base64 编码。

在这个示例中的某个时候，您会看到消息被分成两部分，IV 和密文。初始化向量或 IV 是一个随机值，它被预置到实际加密的消息之前。每次使用 AES 加密消息时，都会生成并使用一个随机值作为加密的一部分。这个随机值被称为一次性号码，简单地意味着只使用一次的数字。

为什么要创建这些一次性值？特别是如果它们不保密，并且直接放在加密消息的前面，它有什么作用？随机 IV 的使用方式类似于盐。它主要用于当相同的消息被重复加密时，每次的密文都是不同的。

要使用**Galois/Counter Mode**（GCM）而不是 CFB，请更改加密和解密方法。GCM 具有更好的性能和效率，因为它允许并行处理。在[`en.wikipedia.org/wiki/Galois/Counter_Mode`](https://en.wikipedia.org/wiki/Galois/Counter_Mode)上了解更多关于 GCM 的信息。

从 AES 密码开始，并调用`cipher.NewCFBEncrypter(block, iv)`。然后根据您是否需要加密或解密，您将使用您生成的 nonce 调用`.Seal()`，或者调用`.Open()`并传递分离的 nonce 和密文：

```go
package main

import (
   "crypto/aes"
   "crypto/cipher"
   "crypto/rand"
   "fmt"
   "io"
   "io/ioutil"
   "os"
   "log"
)

func printUsage() {
   fmt.Printf(os.Args[0] + `

Encrypt or decrypt a file using AES with a 256-bit key file.
This program can also generate 256-bit keys.

Usage:
  ` + os.Args[0] + ` [-h|--help]
  ` + os.Args[0] + ` [-g|--genkey]
  ` + os.Args[0] + ` <keyFile> <file> [-d|--decrypt]

Examples:
  # Generate a 32-byte (256-bit) key
  ` + os.Args[0] + ` --genkey

  # Encrypt with secret key. Output to STDOUT
  ` + os.Args[0] + ` --genkey > secret.key

  # Encrypt message using secret key. Output to ciphertext.dat
  ` + os.Args[0] + ` secret.key message.txt > ciphertext.dat

  # Decrypt message using secret key. Output to STDOUT
  ` + os.Args[0] + ` secret.key ciphertext.dat -d

  # Decrypt message using secret key. Output to message.txt
  ` + os.Args[0] + ` secret.key ciphertext.dat -d > cleartext.txt
`)
}

// Check command-line arguments.
// If the help or generate key functions are chosen
// they are run and then the program exits
// otherwise it returns keyFile, file, decryptFlag.
func checkArgs() (string, string, bool) {
   if len(os.Args) < 2  || len(os.Args) > 4 {
      printUsage()
      os.Exit(1)
   }

   // One arg provided
   if len(os.Args) == 2 {
      // Only -h, --help and --genkey are valid one-argument uses
      if os.Args[1] == "-h" || os.Args[1] == "--help" {
         printUsage() // Print help text
         os.Exit(0) // Exit gracefully no error
      }
      if os.Args[1] == "-g" || os.Args[1] == "--genkey" {
         // Generate a key and print to STDOUT
         // User should redirect output to a file if needed
         key := generateKey()
         fmt.Printf(string(key[:])) // No newline
         os.Exit(0) // Exit gracefully
      }
   }

   // The only use options left is
   // encrypt <keyFile> <file> [-d|--decrypt]
   // If there are only 2 args provided, they must be the
   // keyFile and file without a decrypt flag.
   if len(os.Args) == 3 {
      // keyFile, file, decryptFlag
      return os.Args[1], os.Args[2], false 
   }
   // If 3 args are provided,
   // check that the last one is -d or --decrypt
   if len(os.Args) == 4 {
      if os.Args[3] != "-d" && os.Args[3] != "--decrypt" {
         fmt.Println("Error: Unknown usage.")
         printUsage()
         os.Exit(1) // Exit with error code
      }
      return os.Args[1], os.Args[2], true
   }
    return "", "", false // Default blank return
}

func generateKey() []byte {
   randomBytes := make([]byte, 32) // 32 bytes, 256 bit
   numBytesRead, err := rand.Read(randomBytes)
   if err != nil {
      log.Fatal("Error generating random key.", err)
   }
   if numBytesRead != 32 {
      log.Fatal("Error generating 32 random bytes for key.")
   }
   return randomBytes
}

// AES encryption
func encrypt(key, message []byte) ([]byte, error) {
   // Initialize block cipher
   block, err := aes.NewCipher(key)
   if err != nil {
      return nil, err
   }

   // Create the byte slice that will hold encrypted message
   cipherText := make([]byte, aes.BlockSize+len(message))

   // Generate the Initialization Vector (IV) nonce
   // which is stored at the beginning of the byte slice
   // The IV is the same length as the AES blocksize
   iv := cipherText[:aes.BlockSize]
   _, err = io.ReadFull(rand.Reader, iv)
   if err != nil {
      return nil, err
   }

   // Choose the block cipher mode of operation
   // Using the cipher feedback (CFB) mode here.
   // CBCEncrypter also available.
   cfb := cipher.NewCFBEncrypter(block, iv)
   // Generate the encrypted message and store it
   // in the remaining bytes after the IV nonce
   cfb.XORKeyStream(cipherText[aes.BlockSize:], message)

   return cipherText, nil
}

// AES decryption
func decrypt(key, cipherText []byte) ([]byte, error) {
   // Initialize block cipher
   block, err := aes.NewCipher(key)
   if err != nil {
      return nil, err
   }

   // Separate the IV nonce from the encrypted message bytes
   iv := cipherText[:aes.BlockSize]
   cipherText = cipherText[aes.BlockSize:]

   // Decrypt the message using the CFB block mode
   cfb := cipher.NewCFBDecrypter(block, iv)
   cfb.XORKeyStream(cipherText, cipherText)

   return cipherText, nil
}

func main() {
   // if generate key flag, just output a key to stdout and exit
   keyFile, file, decryptFlag := checkArgs()

   // Load key from file
   keyFileData, err := ioutil.ReadFile(keyFile)
   if err != nil {
      log.Fatal("Unable to read key file contents.", err)
   }

   // Load file to be encrypted or decrypted
   fileData, err := ioutil.ReadFile(file)
   if err != nil {
      log.Fatal("Unable to read key file contents.", err)
   }

   // Perform encryption unless the decryptFlag was provided
   // Outputs to STDOUT. User can redirect output to file.
   if decryptFlag {
      message, err := decrypt(keyFileData, fileData)
      if err != nil {
         log.Fatal("Error decrypting. ", err)
      }
      fmt.Printf("%s", message)
   } else {
      cipherText, err := encrypt(keyFileData, fileData)
      if err != nil {
         log.Fatal("Error encrypting. ", err)
      }
      fmt.Printf("%s", cipherText)
   }
}
```

# 非对称加密

当每个方都有两个密钥时，就是非对称的。每一方都需要一个公钥和私钥对。非对称加密算法包括 RSA，DSA 和 ECDSA。Go 标准库中有 RSA，DSA 和 ECDSA 的包。一些使用非对称加密的应用包括**安全外壳**（**SSH**），**安全套接字层**（**SSL**）和**很好的隐私**（**PGP**）。

SSL 是由网景公司最初开发的**安全套接字层**，版本 2 于 1995 年公开发布。它用于加密服务器和客户端之间的通信，提供机密性，完整性和认证。**TLS**，或**传输层安全**，是 SSL 的新版本，1.2 版于 2008 年作为 RFC 5246 定义。Go 的 TLS 包并未完全实现规范，但实现了主要部分。了解有关 Go 的`crypto/tls`包的更多信息，请访问[`golang.org/pkg/crypto/tls/`](https://golang.org/pkg/crypto/tls/)。

您只能加密小于密钥大小的东西，这通常是 2048 位。由于这种大小限制，非对称 RSA 加密不适用于加密整个文档，这些文档很容易超过 2048 位或 256 字节。另一方面，例如 AES 的对称加密可以加密大型文档，但它需要双方共享的密钥。TLS/SSL 使用非对称和对称加密的组合。初始连接和握手使用每一方的公钥和私钥进行非对称加密。一旦建立连接，将生成并共享一个共享密钥。一旦双方都知道共享密钥，非对称加密将被丢弃，其余的通信将使用对称加密，例如使用共享密钥的 AES。

这里的示例将使用 RSA 密钥。我们将介绍如何生成自己的公钥和私钥，并将它们保存为 PEM 编码的文件，数字签名消息和验证签名。在下一节中，我们将使用这些密钥创建自签名证书并建立安全的 TLS 连接。

# 生成公钥和私钥对

在使用非对称加密之前，您需要一个公钥和私钥对。私钥必须保密并且不与任何人共享。公钥应与他人共享。

**RSA**（**Rivest-Shamir-Adleman**）和**ECDSA**（**椭圆曲线数字签名算法**）算法在 Go 标准库中可用。ECDSA 被认为更安全，但 RSA 是 SSL 证书中最常用的算法。

您可以选择对私钥进行密码保护。您不需要这样做，但这是额外的安全层。由于私钥非常敏感，建议您使用密码保护。

如果要使用对称加密算法（例如 AES）对私钥文件进行密码保护，可以使用一些标准库函数。您将需要的主要函数是`x509.EncryptPEMBlock()`，`x509.DecryptPEMBlock()`和`x509.IsEncryptedPEMBlock()`。

要执行使用 OpenSSL 生成私钥和公钥文件的等效操作，请使用以下内容：

```go
# Generate the private key  
openssl genrsa -out priv.pem 2048 
# Extract the public key from the private key 
openssl rsa -in priv.pem -pubout -out public.pem 
```

您可以在[`golang.org/pkg/encoding/pem/`](https://golang.org/pkg/encoding/pem/)了解有关 Go 的 PEM 编码的更多信息。参考以下代码：

```go
package main

import (
   "crypto/rand"
   "crypto/rsa"
   "crypto/x509"
   "encoding/pem"
   "fmt"
   "log"
   "os"
   "strconv"
)

func printUsage() {
   fmt.Printf(os.Args[0] + `

Generate a private and public RSA keypair and save as PEM files.
If no key size is provided, a default of 2048 is used.

Usage:
  ` + os.Args[0] + ` <private_key_filename> <public_key_filename>       [keysize]

Examples:
  # Store generated private and public key in privkey.pem and   pubkey.pem
  ` + os.Args[0] + ` priv.pem pub.pem
  ` + os.Args[0] + ` priv.pem pub.pem 4096`)
}

func checkArgs() (string, string, int) {
   // Too many or too few arguments
   if len(os.Args) < 3 || len(os.Args) > 4 {
      printUsage()
      os.Exit(1)
   }

   defaultKeySize := 2048

   // If there are 2 args provided, privkey and pubkey filenames
   if len(os.Args) == 3 {
      return os.Args[1], os.Args[2], defaultKeySize
   }

   // If 3 args provided, privkey, pubkey, keysize
   if len(os.Args) == 4 {
      keySize, err := strconv.Atoi(os.Args[3])
      if err != nil {
         printUsage()
         fmt.Println("Invalid keysize. Try 1024 or 2048.")
         os.Exit(1)
      }
      return os.Args[1], os.Args[2], keySize
   }

   return "", "", 0 // Default blank return catch-all
}

// Encode the private key as a PEM file
// PEM is a base-64 encoding of the key
func getPrivatePemFromKey(privateKey *rsa.PrivateKey) *pem.Block {
   encodedPrivateKey := x509.MarshalPKCS1PrivateKey(privateKey)
   var privatePem = &pem.Block {
      Type: "RSA PRIVATE KEY",
      Bytes: encodedPrivateKey,
   }
   return privatePem
}

// Encode the public key as a PEM file
func generatePublicPemFromKey(publicKey rsa.PublicKey) *pem.Block {
   encodedPubKey, err := x509.MarshalPKIXPublicKey(&publicKey)
   if err != nil {
      log.Fatal("Error marshaling PKIX pubkey. ", err)
   }

   // Create a public PEM structure with the data
   var publicPem = &pem.Block{
      Type:  "PUBLIC KEY",
      Bytes: encodedPubKey,
   }
   return publicPem
}

func savePemToFile(pemBlock *pem.Block, filename string) {
   // Save public pem to file
   publicPemOutputFile, err := os.Create(filename)
   if err != nil {
      log.Fatal("Error opening pubkey output file. ", err)
   }
   defer publicPemOutputFile.Close()

   err = pem.Encode(publicPemOutputFile, pemBlock)
   if err != nil {
      log.Fatal("Error encoding public PEM. ", err)
   }
}

// Generate a public and private RSA key in PEM format
func main() {
   privatePemFilename, publicPemFilename, keySize := checkArgs()

   // Generate private key
   privateKey, err := rsa.GenerateKey(rand.Reader, keySize)
   if err != nil {
      log.Fatal("Error generating private key. ", err)
   }

   // Encode keys to PEM format
   privatePem := getPrivatePemFromKey(privateKey)
   publicPem := generatePublicPemFromKey(privateKey.PublicKey)

   // Save the PEM output to files
   savePemToFile(privatePem, privatePemFilename)
   savePemToFile(publicPem, publicPemFilename)

   // Print the public key to STDOUT for convenience
   fmt.Printf("%s", pem.EncodeToMemory(publicPem))
}
```

# 数字签名消息

签署消息的目的是让接收者知道消息来自正确的人。要签署消息，首先生成消息的哈希，然后使用您的私钥加密哈希。加密的哈希就是您的签名。

接收者将解密你的签名以获得你提供的原始哈希，然后他们将对消息进行哈希处理，看看他们自己从消息中生成的哈希是否与签名的解密值匹配。如果匹配，接收者就知道签名是有效的，并且来自正确的发送者。

请注意，签署一条消息实际上并不加密消息。如果需要，你仍然需要在发送消息之前对消息进行加密。如果你想公开发布你的消息，你可能不想加密消息本身。其他人仍然可以使用签名来验证发布消息的人。

只有小于 RSA 密钥大小的消息才能被签名。因为 SHA-256 哈希总是具有相同的输出长度，我们可以确保它在可接受的大小限制内。在这个例子中，我们使用了 RSA PKCS#1 v1.5 标准签名和 SHA-256 哈希方法。

Go 编程语言提供了核心包中的函数来处理签名和验证。主要函数是`rsa.VerifyPKCS1v5`。这个函数负责对消息进行哈希处理，然后用私钥对其进行加密。

以下程序将接收一条消息和一个私钥，并将签名输出到`STDOUT`：

```go
package main

import (
   "crypto"
   "crypto/rand"
   "crypto/rsa"
   "crypto/sha256"
   "crypto/x509"
   "encoding/pem"
   "fmt"
   "io/ioutil"
   "log"
   "os"
)

func printUsage() {
   fmt.Println(os.Args[0] + `

Cryptographically sign a message using a private key.
Private key should be a PEM encoded RSA key.
Signature is generated using SHA256 hash.
Output signature is stored in filename provided.

Usage:
  ` + os.Args[0] + ` <privateKeyFilename> <messageFilename>   <signatureFilename>

Example:
  # Use priv.pem to encrypt msg.txt and output to sig.txt.256
  ` + os.Args[0] + ` priv.pem msg.txt sig.txt.256
`)
}

// Get arguments from command line
func checkArgs() (string, string, string) {
   // Need exactly 3 arguments provided
   if len(os.Args) != 4 {
      printUsage()
      os.Exit(1)
   }

   // Private key file name and message file name
   return os.Args[1], os.Args[2], os.Args[3]
}

// Cryptographically sign a message= creating a digital signature
// of the original message. Uses SHA-256 hashing.
func signMessage(privateKey *rsa.PrivateKey, message []byte) []byte {
   hashed := sha256.Sum256(message)

   signature, err := rsa.SignPKCS1v15(
      rand.Reader,
      privateKey,
      crypto.SHA256,
      hashed[:],
   )
   if err != nil {
      log.Fatal("Error signing message. ", err)
   }

   return signature
}

// Load the message that will be signed from file
func loadMessageFromFile(messageFilename string) []byte {
   fileData, err := ioutil.ReadFile(messageFilename)
   if err != nil {
      log.Fatal(err)
   }
   return fileData
}

// Load the RSA private key from a PEM encoded file
func loadPrivateKeyFromPemFile(privateKeyFilename string) *rsa.PrivateKey {
   // Quick load file to memory
   fileData, err := ioutil.ReadFile(privateKeyFilename)
   if err != nil {
      log.Fatal(err)
   }

   // Get the block data from the PEM encoded file
   block, _ := pem.Decode(fileData)
   if block == nil || block.Type != "RSA PRIVATE KEY" {
      log.Fatal("Unable to load a valid private key.")
   }

   // Parse the bytes and put it in to a proper privateKey struct
   privateKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
   if err != nil {
      log.Fatal("Error loading private key.", err)
   }

   return privateKey
}

// Save data to file
func writeToFile(filename string, data []byte) error {
   // Open a new file for writing only
   file, err := os.OpenFile(
      filename,
      os.O_WRONLY|os.O_TRUNC|os.O_CREATE,
      0666,
   )
   if err != nil {
      return err
   }
   defer file.Close()

   // Write bytes to file
   _, err = file.Write(data)
   if err != nil {
      return err
   }

   return nil
}

// Sign a message using a private RSA key
func main() {
   // Get arguments from command line
   privateKeyFilename, messageFilename, sigFilename := checkArgs()

   // Load message and private key files from disk
   message := loadMessageFromFile(messageFilename)
   privateKey := loadPrivateKeyFromPemFile(privateKeyFilename)

   // Cryptographically sign the message
   signature := signMessage(privateKey, message)

   // Output to file
   writeToFile(sigFilename, signature)
}
```

# 验证签名

在前面的例子中，我们学习了如何为接收者创建一条消息的签名以进行验证。现在让我们来看看验证签名的过程。

如果你收到一条消息和一个签名，你必须首先使用发送者的公钥解密签名。然后对原始消息进行哈希处理，看看你的哈希是否与解密的签名匹配。如果你的哈希与解密的签名匹配，那么你可以确定发送者是拥有与你用来验证的公钥配对的私钥的人。

为了验证签名，我们使用了与创建签名相同的算法（RSA PKCS#1 v1.5 with SHA-256）。

这个例子需要两个命令行参数。第一个参数是创建签名的人的公钥，第二个参数是带有签名的文件。要创建一个签名文件，可以使用前面例子中的签名程序，并将输出重定向到一个文件中。

与前一节类似，Go 语言在标准库中有一个用于验证签名的函数。我们可以使用`rsa.VerifyPKCS1v5()`来比较消息哈希与签名的解密值，并查看它们是否匹配：

```go
package main

import (
   "crypto"
   "crypto/rsa"
   "crypto/sha256"
   "crypto/x509"
   "encoding/pem"
   "fmt"
   "io/ioutil"
   "log"
   "os"
)

func printUsage() {
    fmt.Println(os.Args[0] + `

Verify an RSA signature of a message using SHA-256 hashing.
Public key is expected to be a PEM file.

Usage:
  ` + os.Args[0] + ` <publicKeyFilename> <signatureFilename> <messageFilename>

Example:
  ` + os.Args[0] + ` pubkey.pem signature.txt message.txt
`)
}

// Get arguments from command line
func checkArgs() (string, string, string) {
   // Expect 3 arguments: pubkey, signature, message file names
   if len(os.Args) != 4 {
      printUsage()
      os.Exit(1)
   }

   return os.Args[1], os.Args[2], os.Args[3]
}

// Returns bool whether signature was verified
func verifySignature(
   signature []byte,
   message []byte,
   publicKey *rsa.PublicKey) bool {

   hashedMessage := sha256.Sum256(message)

   err := rsa.VerifyPKCS1v15(
      publicKey,
      crypto.SHA256,
      hashedMessage[:],
      signature,
   )

   if err != nil {
      log.Println(err)
      return false
   }
   return true // If no error, match.
}

// Load file to memory
func loadFile(filename string) []byte {
   fileData, err := ioutil.ReadFile(filename)
   if err != nil {
      log.Fatal(err)
   }
   return fileData
}

// Load a public RSA key from a PEM encoded file
func loadPublicKeyFromPemFile(publicKeyFilename string) *rsa.PublicKey {
   // Quick load file to memory
   fileData, err := ioutil.ReadFile(publicKeyFilename)
   if err != nil {
      log.Fatal(err)
   }

   // Get the block data from the PEM encoded file
   block, _ := pem.Decode(fileData)
   if block == nil || block.Type != "PUBLIC KEY" {
      log.Fatal("Unable to load valid public key. ")
   }

   // Parse the bytes and store in a public key format
   publicKey, err := x509.ParsePKIXPublicKey(block.Bytes)
   if err != nil {
      log.Fatal("Error loading public key. ", err)
   }

   return publicKey.(*rsa.PublicKey) // Cast interface to PublicKey
}

// Verify a cryptographic signature using RSA PKCS#1 v1.5 with SHA-256
// and a PEM encoded PKIX public key.
func main() {
   // Parse command line arguments
   publicKeyFilename, signatureFilename, messageFilename :=   
      checkArgs()

   // Load all the files from disk
   publicKey := loadPublicKeyFromPemFile(publicKeyFilename)
   signature := loadFile(signatureFilename)
   message := loadFile(messageFilename)

   // Verify signature
   valid := verifySignature(signature, message, publicKey)

   if valid {
      fmt.Println("Signature verified.")
   } else {
      fmt.Println("Signature could not be verified.")
   }
}
```

# TLS

我们通常不会使用 RSA 加密整个消息，因为它只能加密小于密钥大小的消息。解决这个问题的方法通常是从使用 RSA 密钥加密的小消息开始通信。当它们建立了一个安全通道后，它们可以安全地交换一个共享密钥，用于对其余消息进行对称加密，而不受大小限制。这是 SSL 和 TLS 用来建立安全通信的方法。握手过程负责协商在生成和共享对称密钥时将使用哪些加密算法。

# 生成自签名证书

要使用 Go 创建自签名证书，你需要一个公钥和私钥对。x509 包中有一个用于创建证书的函数。它需要公钥和私钥以及一个包含所有信息的模板证书。由于我们是自签名的，模板证书也将用作执行签名的父证书。

每个应用程序可以以不同的方式处理自签名证书。有些应用程序会在证书是自签名时警告你，有些会拒绝接受它，而其他一些则会在不警告你的情况下使用它。当你编写自己的应用程序时，你将不得不决定是否要验证证书或接受自签名证书。

重要的函数是`x509.CreateCertificate()`，在[`golang.org/pkg/crypto/x509/#CreateCertificate`](https://golang.org/pkg/crypto/x509/#CreateCertificate)中有引用。以下是函数签名：

```go
func CreateCertificate (rand io.Reader, template, parent *Certificate, pub, 
   priv interface{}) (cert []byte, err error)
```

这个例子将使用私钥生成一个由它签名的证书，并将其保存为 PEM 格式的文件。一旦你创建了一个自签名证书，你就可以将该证书与私钥一起使用，运行安全的 TLS 套接字监听器和 Web 服务器。

为了简洁起见，这个例子将证书所有者信息和主机名 IP 硬编码为 localhost。这对于在本地机器上进行测试已经足够了。

根据需要修改这些内容，自定义值，通过命令行参数输入，或者使用标准输入动态获取用户的值，如下面的代码块所示：

```go
package main

import (
   "crypto/rand"
   "crypto/rsa"
   "crypto/x509/pkix"
   "crypto/x509"
   "encoding/pem"
   "fmt"
   "io/ioutil"
   "log"
   "math/big"
   "net"
   "os"
   "time"
)

func printUsage() {
   fmt.Println(os.Args[0] + ` - Generate a self signed TLS certificate

Usage:
  ` + os.Args[0] + ` <privateKeyFilename> <certOutputFilename> [-ca|--cert-authority]

Example:
  ` + os.Args[0] + ` priv.pem cert.pem
  ` + os.Args[0] + ` priv.pem cacert.pem -ca
`)
}

func checkArgs() (string, string, bool) {
   if len(os.Args) < 3 || len(os.Args) > 4 {
      printUsage()
      os.Exit(1)
   }

   // See if the last cert authority option was passed
   isCA := false // Default
   if len(os.Args) == 4 {
      if os.Args[3] == "-ca" || os.Args[3] == "--cert-authority" {
         isCA = true
      }
   }

   // Private key filename, cert output filename, is cert authority
   return os.Args[1], os.Args[2], isCA
}

func setupCertificateTemplate(isCA bool) x509.Certificate {
   // Set valid time frame to start now and end one year from now
   notBefore := time.Now()
   notAfter := notBefore.Add(time.Hour * 24 * 365) // 1 year/365 days

   // Generate secure random serial number
   serialNumberLimit := new(big.Int).Lsh(big.NewInt(1), 128)
   randomNumber, err := rand.Int(rand.Reader, serialNumberLimit)
   if err != nil {
      log.Fatal("Error generating random serial number. ", err)
   }

   nameInfo := pkix.Name{
      Organization: []string{"My Organization"},
      CommonName: "localhost",
      OrganizationalUnit: []string{"My Business Unit"},
      Country:        []string{"US"}, // 2-character ISO code
      Province:       []string{"Texas"}, // State
      Locality:       []string{"Houston"}, // City
   }

   // Create the certificate template
   certTemplate := x509.Certificate{
      SerialNumber: randomNumber,
      Subject: nameInfo,
      EmailAddresses: []string{"test@localhost"},
      NotBefore: notBefore,
      NotAfter: notAfter,
      KeyUsage: x509.KeyUsageKeyEncipherment |   
         x509.KeyUsageDigitalSignature,
      // For ExtKeyUsage, default to any, but can specify to use
      // only as server or client authentication, code signing, etc
      ExtKeyUsage: []x509.ExtKeyUsage{x509.ExtKeyUsageAny},
      BasicConstraintsValid: true,
      IsCA: false,
   }

   // To create a certificate authority that can sign cert signing   
   // requests, set these
   if isCA {
      certTemplate.IsCA = true
      certTemplate.KeyUsage = certTemplate.KeyUsage |  
         x509.KeyUsageCertSign
   }

   // Add any IP addresses and hostnames covered by this cert
   // This example only covers localhost
   certTemplate.IPAddresses = []net.IP{net.ParseIP("127.0.0.1")}
   certTemplate.DNSNames = []string{"localhost", "localhost.local"}

   return certTemplate
}

// Load the RSA private key from a PEM encoded file
func loadPrivateKeyFromPemFile(privateKeyFilename string) *rsa.PrivateKey {
   // Quick load file to memory
   fileData, err := ioutil.ReadFile(privateKeyFilename)
   if err != nil {
      log.Fatal("Error loading private key file. ", err)
   }

   // Get the block data from the PEM encoded file
   block, _ := pem.Decode(fileData)
   if block == nil || block.Type != "RSA PRIVATE KEY" {
      log.Fatal("Unable to load a valid private key.")
   }

   // Parse the bytes and put it in to a proper privateKey struct
   privateKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
   if err != nil {
      log.Fatal("Error loading private key. ", err)
   }

   return privateKey
}

// Save the certificate as a PEM encoded file
func writeCertToPemFile(outputFilename string, derBytes []byte ) {
   // Create a PEM from the certificate
   certPem := &pem.Block{Type: "CERTIFICATE", Bytes: derBytes}

   // Open file for writing
   certOutfile, err := os.Create(outputFilename)
   if err != nil {
      log.Fatal("Unable to open certificate output file. ", err)
   }
   pem.Encode(certOutfile, certPem)
   certOutfile.Close()
}

// Create a self-signed TLS/SSL certificate for localhost 
// with an RSA private key
func main() {
   privPemFilename, certOutputFilename, isCA := checkArgs()

   // Private key of signer - self signed means signer==signee
   privKey := loadPrivateKeyFromPemFile(privPemFilename)

   // Public key of signee. Self signing means we are the signer and    
   // the signee so we can just pull our public key from our private key
   pubKey := privKey.PublicKey

   // Set up all the certificate info
   certTemplate := setupCertificateTemplate(isCA)

   // Create (and sign with the priv key) the certificate
   certificate, err := x509.CreateCertificate(
      rand.Reader,
      &certTemplate,
      &certTemplate,
      &pubKey,
      privKey,
   )
   if err != nil {
      log.Fatal("Failed to create certificate. ", err)
   }

   // Format the certificate as a PEM and write to file
   writeCertToPemFile(certOutputFilename, certificate)
}
```

# 创建证书签名请求

如果你不想创建自签名证书，你必须创建一个证书签名请求，并让受信任的证书颁发机构对其进行签名。你可以通过调用`x509.CreateCertificateRequest()`并传递一个带有私钥的`x509.CertificateRequest`对象来创建一个证书请求。

使用 OpenSSL 进行等效操作如下：

```go
# Create CSR 
openssl req -new -key priv.pem -out csr.pem 
# View details to verify request was created properly 
openssl req -verify -in csr.pem -text -noout 
```

这个例子演示了如何创建证书签名请求：

```go
package main

import (
   "crypto/rand"
   "crypto/rsa"
   "crypto/x509"
   "crypto/x509/pkix"
   "encoding/pem"
   "fmt"
   "io/ioutil"
   "log"
   "net"
   "os"
)

func printUsage() {
   fmt.Println(os.Args[0] + ` - Create a certificate signing request  
   with a private key.

Private key is expected in PEM format. Certificate valid for localhost only.
Certificate signing request is created using the SHA-256 hash.

Usage:
  ` + os.Args[0] + ` <privateKeyFilename> <csrOutputFilename>

Example:
  ` + os.Args[0] + ` priv.pem csr.pem
`)
}

func checkArgs() (string, string) {
   if len(os.Args) != 3 {
      printUsage()
      os.Exit(1)
   }

   // Private key filename, cert signing request output filename
   return os.Args[1], os.Args[2]
}

// Load the RSA private key from a PEM encoded file
func loadPrivateKeyFromPemFile(privateKeyFilename string) *rsa.PrivateKey {
   // Quick load file to memory
   fileData, err := ioutil.ReadFile(privateKeyFilename)
   if err != nil {
      log.Fatal("Error loading private key file. ", err)
   }

   // Get the block data from the PEM encoded file
   block, _ := pem.Decode(fileData)
   if block == nil || block.Type != "RSA PRIVATE KEY" {
      log.Fatal("Unable to load a valid private key.")
   }

   // Parse the bytes and put it in to a proper privateKey struct
   privateKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
   if err != nil {
      log.Fatal("Error loading private key.", err)
   }

   return privateKey
}

// Create a CSR PEM and save to file
func saveCSRToPemFile(csr []byte, filename string) {
   csrPem := &pem.Block{
      Type:  "CERTIFICATE REQUEST",
      Bytes: csr,
   }
   csrOutfile, err := os.Create(filename)
   if err != nil {
      log.Fatal("Error opening "+filename+" for saving. ", err)
   }
   pem.Encode(csrOutfile, csrPem)
}

// Create a certificate signing request with a private key 
// valid for localhost
func main() {
   // Load parameters
   privKeyFilename, csrOutFilename := checkArgs()
   privKey := loadPrivateKeyFromPemFile(privKeyFilename)

   // Prepare information about organization the cert will belong to
   nameInfo := pkix.Name{
      Organization:       []string{"My Organization Name"},
      CommonName:         "localhost",
      OrganizationalUnit: []string{"Business Unit Name"},
      Country:            []string{"US"}, // 2-character ISO code
      Province:           []string{"Texas"},
      Locality:           []string{"Houston"}, // City
   }

   // Prepare CSR template
   csrTemplate := x509.CertificateRequest{
      Version:            2, // Version 3, zero-indexed values
      SignatureAlgorithm: x509.SHA256WithRSA,
      PublicKeyAlgorithm: x509.RSA,
      PublicKey:          privKey.PublicKey,
      Subject:            nameInfo,

      // Subject Alternate Name values.
      DNSNames:       []string{"Business Unit Name"},
      EmailAddresses: []string{"test@localhost"},
      IPAddresses:    []net.IP{},
   }

   // Create the CSR based off the template
   csr, err := x509.CreateCertificateRequest(rand.Reader,  
      &csrTemplate, privKey)
   if err != nil {
      log.Fatal("Error creating certificate signing request. ", err)
   }
   saveCSRToPemFile(csr, csrOutFilename)
}
```

# 签署证书请求

在前面的例子中，当生成自签名证书时，我们已经演示了创建签名证书的过程。在自签名的例子中，我们只是使用了与签名者相同的证书模板。因此，没有单独的代码示例。唯一的区别是进行签名的父证书或要签名的模板应该被替换为不同的证书。

这是`x509.CreateCertificate()`的函数定义：

```go
func CreateCertificate(rand io.Reader, template, parent *Certificate, pub, 
   priv interface{}) (cert []byte, err error)
```

在自签名的例子中，模板和父证书是同一个对象。要签署证书请求，创建一个新的证书对象，并用签名请求中的信息填充字段。将新证书作为模板，使用签名者的证书作为父证书。`pub`参数是受让人的公钥，`priv`参数是签名者的私钥。签名者是证书颁发机构，受让人是请求者。你可以在[`golang.org/pkg/crypto/x509/#CreateCertificate`](https://golang.org/pkg/crypto/x509/#CreateCertificate)了解更多关于这个函数的信息。

`X509.CreateCertificate()`的参数如下：

+   `rand`：这是密码学安全的伪随机数生成器

+   `template`：这是使用 CSR 中的信息填充的证书模板

+   `parent`：这是签名者的证书

+   `pub`：这是受让人的公钥

+   `priv`：这是签名者的私钥

使用 OpenSSL 进行等效操作如下：

```go
# Create signed certificate using
# the CSR, CA certificate, and private key 
openssl x509 -req -in csr.pem -CA cacert.pem \
-CAkey capriv.pem -CAcreateserial \
-out cert.pem -sha256
# Print info about cert 
openssl x509 -in cert.pem -text -noout  
```

# TLS 服务器

你可以像设置普通套接字连接一样设置监听器，但是加密。只需调用 TLS 的`Listen()`函数，并提供你的证书和私钥。使用前面示例中生成的证书和密钥将起作用。

以下程序将创建一个 TLS 服务器，并回显接收到的任何数据，然后关闭连接。服务器不需要或验证客户端证书，但是用于进行验证的代码被注释掉，以供参考，以防你想要使用证书对客户端进行身份验证：

```go
package main

import (
   "bufio"
   "crypto/tls"
   "fmt"
   "log"
   "net"
   "os"
)

func printUsage() {
   fmt.Println(os.Args[0] + ` - Start a TLS echo server

Server will echo one message received back to client.
Provide a certificate and private key file in PEM format.
Host string in the format: hostname:port

Usage:
  ` + os.Args[0] + ` <certFilename> <privateKeyFilename> <hostString>

Example:
  ` + os.Args[0] + ` cert.pem priv.pem localhost:9999
`)
}

func checkArgs() (string, string, string) {
  if len(os.Args) != 4 {
     printUsage()
     os.Exit(1)
  }

  return os.Args[1], os.Args[2], os.Args[3]
}

// Create a TLS listener and echo back data received by clients.
func main() {
   certFilename, privKeyFilename, hostString := checkArgs()

   // Load the certificate and private key
   serverCert, err := tls.LoadX509KeyPair(certFilename, privKeyFilename)
   if err != nil {
      log.Fatal("Error loading certificate and private key. ", err)
   }

   // Set up certificates, host/ip, and port
   config := &tls.Config{
      // Specify server certificate
      Certificates: []tls.Certificate{serverCert},

      // By default no client certificate is required.
      // To require and validate client certificates, specify the
      // ClientAuthType to be one of:
      //    NoClientCert, RequestClientCert, RequireAnyClientCert,
      //    VerifyClientCertIfGiven, RequireAndVerifyClientCert)

      // ClientAuth: tls.RequireAndVerifyClientCert

      // Define the list of certificates you will accept as
      // trusted certificate authorities with ClientCAs.

      // ClientCAs: *x509.CertPool
   }

   // Create the TLS socket listener
   listener, err := tls.Listen("tcp", hostString, config)
   if err != nil {
      log.Fatal("Error starting TLS listener. ", err)
   }
   defer listener.Close()

   // Listen forever for connections
   for {
      clientConnection, err := listener.Accept()
      if err != nil {
         log.Println("Error accepting client connection. ", err)
         continue
      }
      // Launch a goroutine(thread)go-1.6 to handle each connection
      go handleConnection(clientConnection)
   }
}

// Function that gets launched in a goroutine to handle client connection
func handleConnection(clientConnection net.Conn) {
   defer clientConnection.Close()
   socketReader := bufio.NewReader(clientConnection)
   for {
      // Read a message from the client
      message, err := socketReader.ReadString('\n')
      if err != nil {
         log.Println("Error reading from client socket. ", err)
         return
      }
      fmt.Println(message)

      // Echo back the data to the client.
      numBytesWritten, err := clientConnection.Write([]byte(message))
      if err != nil {
         log.Println("Error writing data to client socket. ", err)
         return
      }
      fmt.Printf("Wrote %d bytes back to client.\n", numBytesWritten)
   }
}
```

# TLS 客户端

TCP 套接字是在网络上进行通信的一种简单而常见的方式。在标准库中使用 Go 的 TLS 层覆盖标准 TCP 套接字非常简单。

客户端拨号 TLS 服务器就像标准套接字一样。客户端通常不需要任何类型的密钥或证书，但服务器可以实现客户端身份验证，并只允许特定用户连接。

该程序将连接到一个 TLS 服务器，并将 STDIN 的内容发送到远程服务器并读取响应。我们可以使用这个程序来测试在上一节中创建的基本 TLS 回显服务器。

在运行此程序之前，请确保上一节中的 TLS 服务器正在运行，以便您可以连接。

请注意，这是一个原始的套接字级服务器。它不是一个 HTTP 服务器。在第九章 *Web 应用*中，有一些运行 HTTPS TLS Web 服务器的示例。

默认情况下，客户端会验证服务器的证书是否由受信任的机构签名。我们必须覆盖这个默认设置，并告诉客户端不要验证证书，因为我们自己签名了它。受信任的证书颁发机构列表是从系统中加载的，但可以通过在`tls.Config`中填充 RootCAs 变量来覆盖。这个例子不会验证服务器证书，但提供了提供一组受信任的 RootCAs 的代码，供参考时注释掉。

您可以通过查看[`golang.org/src/crypto/x509/`](https://golang.org/src/crypto/x509/)中的`root_*.go`文件来了解 Go 如何为每个系统加载证书池。例如，`root_windows.go`和`root_linux.go`加载系统的默认证书。

如果您想连接到服务器并检查或存储其证书，您可以连接，然后检查客户端的`net.Conn.ConnectionState().PeerCertificates`。它以标准的`x509.Certificate`结构形式呈现。要这样做，请参考以下代码块：

```go
package main

import (
   "crypto/tls"
   "fmt"
   "log"
   "os"
)

func printUsage() {
   fmt.Println(os.Args[0] + ` - Send and receive a message to a TLS server

Usage:
  ` + os.Args[0] + ` <hostString>

Example:
  ` + os.Args[0] + ` localhost:9999
`)
}

func checkArgs() string {
   if len(os.Args) != 2 {
      printUsage()
      os.Exit(1)
   }

   // Host string e.g. localhost:9999
   return os.Args[1]
}

// Simple TLS client that sends a message and receives a message
func main() {
   hostString := checkArgs()
   messageToSend := "Hello?\n"

   // Configure TLS settings
   tlsConfig := &tls.Config{
      // Required to accept self-signed certs
      InsecureSkipVerify: true, 
      // Provide your client certificate if necessary
      // Certificates: []Certificate

      // ServerName is used to verify the hostname (unless you are     
      // skipping verification)
      // It is also included in the handshake in case the server uses   
      // virtual hosts Can also just be an IP address 
      // instead of a hostname.
      // ServerName: string,

      // RootCAs that you are willing to accept
      // If RootCAs is nil, the host's default root CAs are used
      // RootCAs: *x509.CertPool
   }

   // Set up dialer and call the server
   connection, err := tls.Dial("tcp", hostString, tlsConfig)
   if err != nil {
      log.Fatal("Error dialing server. ", err)
   }
   defer connection.Close()

   // Write data to socket
   numBytesWritten, err := connection.Write([]byte(messageToSend))
   if err != nil {
      log.Println("Error writing to socket. ", err)
      os.Exit(1)
   }
   fmt.Printf("Wrote %d bytes to the socket.\n", numBytesWritten)

   // Read data from socket and print to STDOUT
   buffer := make([]byte, 100)
   numBytesRead, err := connection.Read(buffer)
   if err != nil {
      log.Println("Error reading from socket. ", err)
      os.Exit(1)
   }
   fmt.Printf("Read %d bytes to the socket.\n", numBytesRead)
   fmt.Printf("Message received:\n%s\n", buffer)
}
```

# 其他加密包

以下部分没有源代码示例，但值得一提。这些由 Go 提供的包是建立在前面示例中演示的原则之上的。

# OpenPGP

PGP 代表**相当好的隐私**，而 OpenPGP 是标准 RFC 4880。PGP 是一个方便的套件，用于加密文本、文件、目录和磁盘。所有原则都与前一节中讨论的 SSL 和 TLS 密钥/证书相同。加密、签名和验证都是一样的。Go 提供了一个 OpenPGP 包。在[`godoc.org/golang.org/x/crypto/openpgp`](https://godoc.org/golang.org/x/crypto/openpgp)上阅读更多关于它的信息。

# 离线记录（OTR）消息

**离线记录**或**OTR**消息是一种端到端加密，用户可以加密他们在任何消息媒介上的通信。这很方便，因为你可以在任何协议上实现加密层，即使协议本身是未加密的。例如，OTR 消息可以在 XMPP、IRC 和许多其他聊天协议上运行。许多聊天客户端，如 Pidgin、Adium 和 Xabber，都支持 OTR，无论是本地支持还是通过插件支持。Go 提供了一个用于实现 OTR 消息的包。在[`godoc.org/golang.org/x/crypto/otr/`](https://godoc.org/golang.org/x/crypto/otr/)上阅读更多关于 Go 的 OTR 支持的信息。

# 总结

阅读完本章后，您应该对 Go 密码包的功能有很好的理解。使用本章中提供的示例作为参考，您应该能够轻松地执行基本的哈希操作、加密、解密、生成密钥和使用密钥。

此外，您应该了解对称加密和非对称加密之间的区别，以及它与哈希的不同之处。您应该对运行 TLS 服务器和与 TLS 客户端连接的基础有所了解。

请记住，目标不是记住每一个细节，而是记住有哪些选项可供选择，以便您可以选择最适合工作的工具。

在下一章中，我们将讨论使用安全外壳（也称为 SSH）。首先介绍了使用公钥和私钥对以及密码进行身份验证，以及如何验证远程主机的密钥。我们还将介绍如何在远程服务器上执行命令以及如何创建交互式外壳。安全外壳利用了本章讨论的加密技术。这是加密的最常见和实用的应用之一。继续阅读，了解更多关于在 Go 中使用 SSH 的内容。
