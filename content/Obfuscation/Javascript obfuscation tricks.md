
## Functions reading themselves
### Comments are included when a function reads itself

Which lets you include arbitrary string data inside a comment and then use it in your program.

*Most* websites that disallow obfuscated code will likely not have automated tools that check for this.

Some deobfuscators unknowingly remove comments, permanently breaking the reversed output (As they are missing vital pieces)

Source: [doctor8296](https://github.com/MichaelXF/js-confuser/issues/151#issue-2640912992)
```js
(function func() {

    // foo = "Y29uc29sZS5sb2coJ2hlbGxvLCB3b3JsZCEnKQ"

	<!-- html comments also work

    func.constructor(atob((''+func).match(/=\s"(\w+)"/)[1]))();
    
})()
```


## Reading stacktraces

### Using line numbers for string decryption
Encryption keys based on the column/line numbers of the code. When someone deobfuscates the code, they would get changed (Formatted, Some functions get removed, Expressions expanded, etc)

If the the unused function dummyFunction is removed then the string fails to be decrypted and you get "V{rrq>Iqlrz" instead.

Error parsing and obtaining the accurate column/line numbers may be difficult.

Source: [MichaelXF](https://github.com/MichaelXF/js-confuser/issues/151#issuecomment-2466844872)
```js
function dummyFunction() {}

function testMe() {
  var error;
  try {
    throw new Error();
  } catch (e) {
    error = e;
  }

  // Parse error stack trace
  var stackTraceLine = error.stack.split("\n")[1];
  var functionName = stackTraceLine.match(/at (.*) \(/)[1];
  var fileAndLocation = stackTraceLine.match(/\((.*)\)/)[1];
  var [file, lineNumber, columnNumber] = fileAndLocation.split(":");

  var stringDecryptionKey = parseInt(lineNumber) + parseInt(columnNumber); // 6 + 11 = 17

  console.log(functionName, lineNumber, columnNumber, stringDecryptionKey); // testMe 6 11 17

  var decryptedString = decrypt("Yt}}~1F~c}u", stringDecryptionKey);

  console.log(decryptedString); // "Hello World"
}

testMe();

// Simple XOR encryption/decryption
function encrypt(text, key) {
  let encryptedText = "";
  for (let i = 0; i < text.length; i++) {
    // XOR the character code with the key
    encryptedText += String.fromCharCode(text.charCodeAt(i) ^ key);
  }
  return encryptedText;
}

function decrypt(encryptedText, key) {
  let decryptedText = "";
  for (let i = 0; i < encryptedText.length; i++) {
    // XOR again with the same key to get the original character
    decryptedText += String.fromCharCode(encryptedText.charCodeAt(i) ^ key);
  }
  return decryptedText;
}
```