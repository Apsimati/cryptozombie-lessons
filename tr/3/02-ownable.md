---
title: Sahip Olunabilir Kontratlar
actions: ['cevapKontrol', 'ipuçları']
requireLogin: true
material:
  editor:
    language: sol
    startingCode:
      "zombiefactory.sol": |
        pragma solidity ^0.4.19;

        // 1. Buraya aktar

        // 2. Burada kalıt al:
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
      "ownable.sol": |
        /**
         * @title Sahip Olunabilir
         * @dev Sahip Olunabilir kontratın bir sahip adresi vardır temel yetkilendirme kontrol fonksiyonları
         * sağlar, bu, "kulanıcı izinlerinin" uygulamasını basitleştirir.
         */
        contract Ownable {
          address public owner;

          event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

          /**
           * @dev Sahip olunabilir yapıcı orijinal kontrat `owner`'ı gönderici hesaba 
           * ayarlar.
           */
          function Ownable() public {
            owner = msg.sender;
          }  


          /**
           * @dev Sahibinden başka herhangi bir hesap tarafından çağrılırsa atılır.
           */
          modifier onlyOwner() {
            require(msg.sender == owner);
            _;
          }


          /** 
           * @dev Kontratın bir newOwner'a transfer kontrolu için geçerli kullanıcıya izin verir.
           * @param newOwner Sahiplik transferi yapılacak adres.
           */
          function transferOwnership(address newOwner) public onlyOwner {
            require(newOwner != address(0));
            OwnershipTransferred(owner, newOwner);
            owner = newOwner;
          }

        }
    answer: >
      pragma solidity ^0.4.19;

      import "./ownable.sol";

      contract ZombieFactory is Ownable {

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
---

Önceki bölümdeki güvenlik açığını fark ettiniz miz?

`setKittyContractAddress` `harici`'dir, yani herhangi biri onu çağırabilir! Bu, fonksiyonu çağıran kimse CryptoKitties kontrat adresini değiştirebilir ve uygulamamızı kendi kullanıcıları için kırabilirler anlamına gelir.

Yeterliğin kontratımızda bu adresi güncellemesini istiyoruz ama herkesin onu güncelleyebilmesini istemiyoruz.

Bunun gibi durumları halletmek için, meydana çıkmış ortak bir uygulama kontratları `Sahip Olunabilir` yapmaktır — onların özel ayrıcalıkları olan bir sahiplerinin (siz) olduğu anlamına geliyor.

## OpenZeppelin'in `Sahip Olunabilir` kontratı

Aşağıdaki **_OpenZeppelin_** Solidity kütüphanesinden alınmış `Sahip Olunabilir` kontrattır. OpenZeppelin, kendi DApps'inizde kullanabileceğiniz güvenlik ve topluluk incelemeli bir kütüphanedir. Bu dersten sonra, Ders 4'ün çıkarılmasını iple çekerken, öğreniminizi ilerletmek için sitelerini kontrol etmenizi önemle tavsiye ederiz!

Aşağıdaki kontratı baştan aşağı okuyun. Henüz öğrenmediğimiz birkaç şey göreceksiniz ama endişelenmeyin, onlardan sonra bahsedeceğiz.

```
/**
 * @title Sahip Olunabilir
 * @dev Sahip Olunabilir kontratın bir sahip adresi vardır temel yetkilendirme kontrol fonksiyonları
 * sağlar, bu, "kulanıcı izinlerinin" uygulamasını basitleştirir.
 */
contract Ownable {
  address public owner;
  event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

  /**
   * @dev Sahip olunabilir yapıcı orijinal kontrat `owner`'ı gönderici hesaba 
   * ayarlar.
   */
  function Ownable() public {
    owner = msg.sender;
  }

  /**
   * @dev Sahibinden başka herhangi bir hesap tarafından çağrılırsa atılır.
   */
  modifier onlyOwner() {
    require(msg.sender == owner);
    _;
  }

  /**
   * @dev Kontratın bir newOwner'a transfer kontrolu için geçerli kullanıcıya izin verir.
   * @param newOwner Sahiplik transferi yapılacak adres.
   */
  function transferOwnership(address newOwner) public onlyOwner {
    require(newOwner != address(0));
    OwnershipTransferred(owner, newOwner);
    owner = newOwner;
  }
}
```

Daha önce görmediğimiz birkaç şey burada:

- Yapıcılar: `function Ownable()` kontrat ile aynı isme sahip opsiyonel bir özel fonksiyon olan bir **_yapıcıdır_**. Kontrat ilk oluşturulduğunda, sadece bir kere gerçekleştirilecektir.
- Fonksiyon Değiştiriciler: `modifier onlyOwner()`. Değiştiriciler diğer fonksiyoları değiştirmek için kullanılan yarı fonksiyon türleridir, çoğunlukla bazı öncelikli gereklilikleri uygulamak için  kullanılırlar. Bu durumda, `onlyOwner` erişimi sınırlamak için kullanılabilir böylelikle **sadece** kontrat **sahibi** bu fonksiyonu çalıştırabilir. Sonraki bölümde fonksiyon değiştiriciler ve garip şeyin `_;` ne yaptığı hakkında daha fazla konuşacağız.
- `indexed` anahtar kelimesi: bunun hakkında endişelenme, ona henüz ihtiyacımız yok.

Yani `Sahip Olunabilir` kontrat temel olarak aşağıdakileri yapar:

1. Bir kontrat oluşturulduğunda, yapıcısı `owner`'ı `msg.sender`'a ayarlar (onu yayan kişi)

2. Belirli fonksiyona sadece `owner` için erişim sınırlayabilecek bir `onlyOwner` ekler

3. Kontrarı yeni bir `owner`a transfer etmenize izin verir
 
`onlyOwner` çoğu Solidity DApps'in bu `Ownable` kontratın bir kopyala/yapıştırı ile başlattığı kontratlar için müşterek bir ihtiyaç gibidir ve sonra ilk kontratı miras olarak ondan alır.

`setKittyContractAddress`'i `onlyOwner`'a sınırlamak istediğimizde, kontratımız için aynısını yapacağız.

## Teste koy

Devam edip `Ownable` kontrat kodunu yeni bir dosyaya, `ownable.sol`'a kopyalamıştık. Devam edelim ve `ZombieFactory`'yi ondan miras olarak alan yapalım.

1. Kodumuzu `ownable.sol` içeriğini `import` etmek için değiştirin. Bunun nasıl yapıldığını hatırlamıyorsanız `zombiefeeding.sol`'a bir göz atın.

2. `ZombieFactory`'yi `Ownable`'dan miras alan olarak değiştirin. Tekrar, bunun nasıl yapıldığını hatırlamıyorsanız `zombiefeeding.sol`'a göz atabilirsiniz.
