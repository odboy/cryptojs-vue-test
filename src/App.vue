<script setup lang="ts">
import { computed, ref } from "vue";
import HelloWorld from "./components/HelloWorld.vue";
import CryptoJS from "crypto-js";
import cjsBase64 from "crypto-js/enc-base64";

function aesDecryptString(encryptedDataB64: string, keyHex: string) {
  let result: string = "";
  if (encryptedDataB64.length > 0) {
    try {
      let decryptout = CryptoJS.AES.decrypt(
        encryptedDataB64,
        CryptoJS.enc.Hex.parse(keyHex),
        {
          mode: CryptoJS.mode.ECB,
          padding: CryptoJS.pad.ZeroPadding,
        }
      );
      result = CryptoJS.enc.Utf8.stringify(decryptout);
    } catch (e) {
      console.log(e);
    }
  }
  return result;
}

function aesEncryptString(plaintextUtf8: string, keyHex: string) {
  let encryptout = CryptoJS.AES.encrypt(
    CryptoJS.enc.Utf8.parse(plaintextUtf8),
    CryptoJS.enc.Hex.parse(keyHex),
    {
      mode: CryptoJS.mode.ECB,
      padding: CryptoJS.pad.ZeroPadding,
    }
  );

  return CryptoJS.enc.Base64.stringify(encryptout.ciphertext);
}

const cipher = ref("1234567890abcdef1234567890abcdef");
const plaintext = ref("");
const ciphertext = ref("");
const plaintext2 = computed(() => {
  return aesDecryptString(ciphertext.value, cipher.value);
});

const encrypt = (s) => {
  ciphertext.value = aesEncryptString(plaintext.value, cipher.value);
};

function ArrayBufferToWordArray(arrayBuffer: ArrayBuffer) {
  console.log("arrayBuffer.byteLength: ",arrayBuffer.byteLength);
  const u8 = new Uint8Array(arrayBuffer, 0, arrayBuffer.byteLength);
  const len = u8.length;
  const words = [];
  for (let i = 0; i < len; i += 1) {
    words[i >>> 2] |= (u8[i] & 0xff) << (24 - (i % 4) * 8);
  }
  return CryptoJS.lib.WordArray.create(words);
}

function wordArrayToUint8Array(wordArray) {
  var arrayOfWords = wordArray.hasOwnProperty("words") ? wordArray.words : [];
  var length = wordArray.hasOwnProperty("sigBytes")
    ? wordArray.sigBytes
    : arrayOfWords.length * 4;
  var uInt8Array = new Uint8Array(length),
    index = 0,
    word,
    i;
  for (i = 0; i < length; i++) {
    word = arrayOfWords[i];
    uInt8Array[index++] = word >> 24;
    uInt8Array[index++] = (word >> 16) & 0xff;
    uInt8Array[index++] = (word >> 8) & 0xff;
    uInt8Array[index++] = word & 0xff;
  }
  return uInt8Array;
}

function wordArrayToArrayBuffer(wordArray) {
  var arrayBuffer = new ArrayBuffer(wordArray.sigBytes);
  var uint8View = new Uint8Array(arrayBuffer);

  for (var i = 0; i < wordArray.sigBytes; i++) {
    uint8View[i] = (wordArray.words[i >>> 2] >>> (24 - (i % 4) * 8)) & 0xff;
  }

  return arrayBuffer;
}
/// 使用ArrayBufferToWordArray
// function aesEncryptFile(originalfile: File, keyHex: string) {
//   const reader = new FileReader();
//   reader.readAsArrayBuffer(originalfile);

//   reader.onload = (e) => {
//     // console.log(e.target.result);
//     // console.log("originalfile: ", ArrayBufferToWordArray(e.target.result));

//     cipherFile = CryptoJS.AES.encrypt(
//       ArrayBufferToWordArray(e.target.result),
//       CryptoJS.enc.Hex.parse(keyHex),
//       {
//         mode: CryptoJS.mode.ECB,
//         padding: CryptoJS.pad.ZeroPadding,
//       }
//     );
//     cipherfileb64.value = CryptoJS.enc.Base64.stringify(cipherFile.ciphertext);
//   };
// }

/// 使用CryptoJS.lib.WordArray.create
function aesEncryptFile(originalfile: File, keyHex: string) {
  const reader = new FileReader();
  reader.readAsArrayBuffer(originalfile);

  reader.onload = () => {
    console.log(reader.result);
    // console.log("originalfile: ", ArrayBufferToWordArray(e.target.result));

    cipherFile = CryptoJS.AES.encrypt(
      CryptoJS.lib.WordArray.create(reader.result),
      CryptoJS.enc.Hex.parse(keyHex),
      {
        mode: CryptoJS.mode.ECB,
        padding: CryptoJS.pad.Pkcs7,
      }
    );
    cipherfileb64.value = CryptoJS.enc.Base64.stringify(cipherFile.ciphertext).slice(0,256);
  };
}

function aesDecryptFile(cipherfile: CryptoJS.WordArray, keyHex: string) {
  return CryptoJS.AES.decrypt(cipherfile, CryptoJS.enc.Hex.parse(keyHex), {
    mode: CryptoJS.mode.ECB,
    padding: CryptoJS.pad.Pkcs7,
  });
}

const originalFile = ref({});
let cipherFile: CryptoJS.WordArray;
const cipherfileb64 = ref("");

const chooseFile = (event) => {
  // console.log(event);
  let file = event.target.files;
  // for (let i = 0; i<file.length; i++){
  //   console.log(file[i].name,file[i].type, file[i].size);
  // }
  originalFile.value = file[0];

  aesEncryptFile(file[0], cipher.value);
};

const downloadFile = () => {
  try {
    let filedata = aesDecryptFile(cipherFile, cipher.value);

    console.log(filedata);
    var element = document.createElement("a");

    let blobdata = new Blob([wordArrayToUint8Array(filedata)]);
    let dataURL = window.URL.createObjectURL(blobdata);
    element.setAttribute("href", dataURL);
    element.setAttribute("download", originalFile.value.name);
    element.style.display = "none";
    document.body.appendChild(element);

    element.click();

    document.body.removeChild(element);
  } catch (error) {
    console.log("解密失败\n", error)
  }
};
</script>

<template>
  <div class="cryptoviews">
    <!-- 字符串加解密 -->
    <div class="cryptoview">
      <h3>AES 字符串</h3>
      <div>
        <text>密钥hex</text>
        <input type="text" v-model="cipher" />
      </div>

      <div>
        <text>原始明文</text>
        <input
          type="text"
          v-model="plaintext"
          placeholder="明文"
          @input="encrypt"
        />
      </div>

      <div>
        <text>密文b64</text>
        <input type="text" v-model="ciphertext" placeholder="密文" />
      </div>
      <div>
        <text>解密明文</text
        ><input type="text" v-model="plaintext2" placeholder="解密结果" />
      </div>
    </div>
    <!-- 文件加解密 -->
    <div class="cryptoview">
      <h3>AES 文件</h3>
      <div>
        <text>密钥hex</text>
        <input type="text" v-model="cipher" />
      </div>

      <div>
        <text>原始文件</text>
        <input
          type="file"
          ref="choosedFile"
          placeholder="选择你的文件"
          @change="chooseFile($event)"
        />
      </div>

      <div>
        <text>密文b64</text>
        <text style="font-size: 5px">前256位</text>
        <input type="text" v-model="cipherfileb64" placeholder="密文" />
      </div>
      <div>
        <text>解密文件</text>
        <button :disabled="false" :loading="true" @click="downloadFile">
          下载
        </button>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.cryptoviews {
  display: flex;
  flex-wrap: wrap;
  .cryptoview {
    display: flex;
    flex-direction: column;
    padding: 0 20px;
    align-items: start;
    input {
      margin: 5px 10px;
      width: 300px;
      height: 30px;
    }
  }
}

.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
