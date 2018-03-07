---
title: Kontratların Değişmezliği
actions: ['cevapKontrol', 'ipuçları']
requireLogin: true
material:
  editor:
    language: sol
    startingCode:
      "zombiefeeding.sol": |
        pragma solidity ^0.4.19;

        import "./zombiefactory.sol";

        contract KittyInterface {
          function getKitty(uint256 _id) external view returns (
            bool isGestating,
            bool isReady,
            uint256 cooldownIndex,
            uint256 nextActionAt,
            uint256 siringWithId,
            uint256 birthTime,
            uint256 matronId,
            uint256 sireId,
            uint256 generation,
            uint256 genes
          );
        }

        contract ZombieFeeding is ZombieFactory {

          // 1. Bunu kaldır:
          address ckAddress = 0x06012c8cf97BEaD5deAe237070F9587f8E7A266d;
          // 2. Bunu bir bildiriye değiştirin:
          KittyInterface kittyContract = KittyInterface(ckAddress);

          // 3. Buraya setKittyContractAddress yöntemi ekleyin

          function feedAndMultiply(uint _zombieId, uint _targetDna, string _species) public {
            require(msg.sender == zombieToOwner[_zombieId]);
            Zombie storage myZombie = zombies[_zombieId];
            _targetDna = _targetDna % dnaModulus;
            uint newDna = (myZombie.dna + _targetDna) / 2;
            if (keccak256(_species) == keccak256("kitty")) {
              newDna = newDna - newDna % 100 + 99;
            }
            _createZombie("NoName", newDna);
          }

          function feedOnKitty(uint _zombieId, uint _kittyId) public {
            uint kittyDna;
            (,,,,,,,,,kittyDna) = kittyContract.getKitty(_kittyId);
            feedAndMultiply(_zombieId, kittyDna, "kitty");
          }

        }
      "zombiefactory.sol": |
        pragma solidity ^0.4.19;

        contract ZombieFactory {

            event NewZombie(uint zombieId, string name, uint dna);

            uint dnaDigits = 16;
            uint dnaModulus = 10 ** dnaDigits;

            struct Zombie {
                string name;
                uint dna;
            }

            Zombie[] public zombies;

            mapping (uint => address) public zombieToOwner;
            mapping (address => uint) ownerZombieCount;

            function _createZombie(string _name, uint _dna) internal {
                uint id = zombies.push(Zombie(_name, _dna)) - 1;
                zombieToOwner[id] = msg.sender;
                ownerZombieCount[msg.sender]++;
                NewZombie(id, _name, _dna);
            }

            function _generateRandomDna(string _str) private view returns (uint) {
                uint rand = uint(keccak256(_str));
                return rand % dnaModulus;
            }

            function createRandomZombie(string _name) public {
                require(ownerZombieCount[msg.sender] == 0);
                uint randDna = _generateRandomDna(_name);
                randDna = randDna - randDna % 100;
                _createZombie(_name, randDna);
            }

        }
    answer: >
      pragma solidity ^0.4.19;

      import "./zombiefactory.sol";

      contract KittyInterface {
        function getKitty(uint256 _id) external view returns (
          bool isGestating,
          bool isReady,
          uint256 cooldownIndex,
          uint256 nextActionAt,
          uint256 siringWithId,
          uint256 birthTime,
          uint256 matronId,
          uint256 sireId,
          uint256 generation,
          uint256 genes
        );
      }

      contract ZombieFeeding is ZombieFactory {

        KittyInterface kittyContract;

        function setKittyContractAddress(address _address) external {
          kittyContract = KittyInterface(_address);
        }

        function feedAndMultiply(uint _zombieId, uint _targetDna, string _species) public {
          require(msg.sender == zombieToOwner[_zombieId]);
          Zombie storage myZombie = zombies[_zombieId];
          _targetDna = _targetDna % dnaModulus;
          uint newDna = (myZombie.dna + _targetDna) / 2;
          if (keccak256(_species) == keccak256("kitty")) {
            newDna = newDna - newDna % 100 + 99;
          }
          _createZombie("NoName", newDna);
        }

        function feedOnKitty(uint _zombieId, uint _kittyId) public {
          uint kittyDna;
          (,,,,,,,,,kittyDna) = kittyContract.getKitty(_kittyId);
          feedAndMultiply(_zombieId, kittyDna, "kitty");
        }

      }
---

Şu ana kadar, Solidity JavaScript gibi diğer dillere oldukça benzer gözüktü. Ancak Ethereum DApps'in normal uygulamalardan oldukça farklı olduğu çeşitli yollar var.

İlk iş olarak, Ethereum'a bir kontrat açtıktan sonra, bu, hiçbir zaman değiştirilemez veya tekrar güncellenemez anlamında, **_sabittir_**.

Kontrata uygulayacağınız başlangıç kodu blok zinciri üzerinde kalıcı olarak orada durur. Bu, Solidity'de güvenliğin bir endişe olmasının bir nedenidir.  Kontrat kodunuzda bir kusur varsa, onu daha sonra yama yapmanın bir yolu yoktur. Kullanıcılarınıza düzeltilmiş farklı bir akıllı kontrat adresi kullanarak başlatmasını söylemek zorunda olacaktınız.

Fakat bu ayrıca akıllı kontratın bir özelliğidir. Kod kuraldır. Bir akıllı kontratın kodunu okursanız ve doğrularsanız, bir fonksiyonu her çağırdığınızda kodun yapacağını söylediği şeyin aynısını her zaman yapacağından emin olabilirsiniz. Hiç kimse daha sonra bu fonksiyonu değiştiremez ve size beklenmedik sonuçlar veremez.

## Harici bağlılıklar

Ders 2'de, CryptoKitties kontrat adresini DApp'imizin içine doğrudan kodladık. Peki CryptoKitties kontratında bir hata oluşsa ve birisi tüm kittiyleri yok etse ne olurdu?

Bu olasılık dışı, fakat bu olmuş olsa DApp'imizi kullanılmaz hale getirirdi — DApp'imiz artık herhangi bir kitty getirmeyen doğrudan kodlanmış bir adrese işaret edecekti. Zombilerimizin kittyleri besleme için gücü yetmeyecekti ve kontratımızı düzeltmek için değiştiremeyecektik.

Bu nedenle, DApp'in anahtar kısımlarını güncellemeye izin verecek fonksiyonların olması çoğu kez mantıklıdır.

Örneğin, DApp'imizin içine CryptoKitties kontrat adresinin doğrudan kodlamak yerine, belki de gelecekte CryptoKitties kontratına bir şey olduğu durumda bu adresi değiştirmemize izin verecek bir `setKittyContractAddress` fonksiyonumzun olması gerekir.

## Teste koy

CryptoKitties kontrat adresini değiştirebilmekiçin Ders 2'den kodumuzu güncelleyelim.

1. Doğrudan kodladığımız `ckAddress` kod satırını silin.

2. Değişken ifade etmek için oluşturduğumuz `kittyContract` satırını değiştirin — örn.hiç bir şeye eşitlemeyin.

3. `setKittyContractAddress` denilen bir fonksiyon oluşturun. Bir argüman alacaktır, `_address` (bir `address`) ve bir `external` fonksiyon olmalıdır.

4. Fonksiyon içine, `kittyContract`'ı `KittyInterface(_address)`'e eşitleyen bir kod satırı ekleyin.

> Not: Bu fonksiyonla bir güvenlik açığı fark ederseniz, endişelenmeyin — onu sonraki bölümde düzelteceğiz ;)
