# Mail::Gpg [![Build Status](https://travis-ci.org/jkraemer/mail-gpg.png?branch=master)](https://travis-ci.org/jkraemer/mail-gpg)

This gem adds GPG/MIME encryption capabilities to the [Ruby Mail
Library](https://github.com/mikel/mail)

For maximum interoperability the gem also supports *decryption* of messages using the non-standard 'PGP-Inline' method
as for example supported in the Mozilla Enigmail OpenPGP plugin.

There may still be GPG encrypted messages that can not be handled by the library, as there are some legacy formats used in the
wild as described in this [Know your PGP implementation](http://binblog.info/2008/03/12/know-your-pgp-implementation/) blog.


## Installation

Add this line to your application's Gemfile:

    gem 'mail-gpg'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install mail-gpg


## Usage

### Encrypting / Signing
Construct your Mail object as usual and specify you want it to be encrypted
with the gpg method:

    Mail.new do
      to 'jane@doe.net'
      from 'john@doe.net'
      subject 'gpg test'
      body "encrypt me!"
      add_file "some_attachment.zip"

      # encrypt message, no signing
      gpg encrypt: true

      # encrypt and sign message with sender's private key, using the given
      # passphrase to decrypt the key
      gpg encrypt: true, sign: true, password: 'secret'

      # encrypt and sign message using a different key
      gpg encrypt: true, sign_as: 'joe@otherdomain.com', password: 'secret'


      # encrypt and sign message and use a callback function to provide the
      # passphrase.
      gpg encrypt: true, sign_as: 'joe@otherdomain.com',
          passphrase_callback: ->(obj, uid_hint, passphrase_info, prev_was_bad, fd){puts "Enter passphrase for #{passphrase_info}: "; (IO.for_fd(fd, 'w') << readline.chomp).flush }
    end.deliver


Make sure all recipients' public keys are present in your local gpg keychain.
You will get errors in case encryption is not possible due to missing keys.
If you collect public key data from your users, you can specify the ascii
armored key data for recipients using the `:keys` option like this:

    johns_key = <<-END
    -----BEGIN PGP PUBLIC KEY BLOCK-----
    Version: GnuPG v1.4.12 (GNU/Linux)

    mQGiBEk39msRBADw1ExmrLD1OUMdfvA7cnVVYTC7CyqfNvHUVuuBDhV7azs
    ....
    END

    Mail.new do
      to 'john@foo.bar'
      gpg encrypt: true, keys: { 'john@foo.bar' => johns_key }
    end

The key will then be imported before actually trying to encrypt/send the mail.
In theory you only need to specify the key once like that, however doing it
every time does not hurt as gpg is clever enough to recognize known keys, only
updating it's db when necessary.

You may also want to have a look at the [GPGME](https://github.com/ueno/ruby-gpgme) docs and code base for more info on the various options, especially regarding the `passphrase_callback` arguments.


### Decrypting

Receive the mail as usual. Check if it is encrypted using the `encrypted?` method. Get a decrypted version of the mail with the `decrypt` method:

```ruby
mail = Mail.first
mail.subject # subject is never encrypted
if mail.encrypted?
  # decrypt using your private key, protected by the given passphrase
  plaintext_mail = mail.decrypt(:password => 'abc')
  # the plaintext_mail, is a full Mail::Message object, just decrypted
end
```

A `GPGME::Error::BadPassphrase` will be raised if the password for the private key is incorrect.
A `EncodingError` will be raised if the encrypted mails is not encoded correctly as a [RFC 3156](http://www.ietf.org/rfc/rfc3156.txt) message.


### Signing only

Just leave the the `:encrypt` option out or pass `encrypt: false`, i.e.


    Mail.new do
      to 'jane@doe.net'
      gpg sign: true
    end.deliver 



## Rails / ActionMailer integration

    class MyMailer < ActionMailer::Base
      default from: 'baz@bar.com'
      def some_mail
        mail to: 'foo@bar.com', subject: 'subject!', gpg: { encrypt: true }
      end
    end

The gpg option takes the same arguments as outlined above for the
Mail::Message#gpg method.

## Running the tests

    bundle exec rake

Test cases use a mock gpghome located in `test/gpghome` in order to not mess
around with your personal gpg keychain.


## Todo

* signature verification for received mails
* on the fly import of recipients' keys from public key servers based on email address or key id
* handle encryption errors due to missing keys - maybe return a list of failed
  recipients
* add some setup code to help initialize a basic keychain directory with
  public/private keypair.


## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request


