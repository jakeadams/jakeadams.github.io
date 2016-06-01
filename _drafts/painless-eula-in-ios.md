---
layout: post
title:  "Painless EULA for iOS"
categories: eula
---
Eventually, someone will ask you to add a EULA to your app. Managing them is annoying. Presenting them to the user every time is unacceptable. Even presenting the EULA with every app update is less than ideal.

The user should only see the EULA when it changes. How do we know when it changes, though? I'll show you how we solved the problem on our app.

Before going on, I was inspired to write this post after reading Donn Felker's excellent post on [handling EULA's on Android](http://www.donnfelker.com/android-a-simple-eula-for-your-android-apps/). Give it a read and compare our implementations and pick the one that works best for you!

The basic concept behind both my and Donn's solution is the same.

* Uniquely identify each EULA version
* If the user accepts the EULA, then store that identifier in `NSUserDefaults`
* Each time you launch the app, compare the stored identifier to the current EULA's identifier
* If they differ, present the EULA

EULA's aren't exactly fun and if us developers had our way, we wouldn't have one. 
All I really wanted to have to do was update the EULA file with whatever HTML the legal team gave us and be done with it. 
I had an epiphany at just the right moment (which is rare for me. Usually epiphanies happen *after* I've shipped): use a SHA hash as the identifier.

The concept is similar to downloading large files and comparing the hash from the source to the hash of the file you downloaded. If they match, then you've got a good download. Similarly, if the content of the EULA changes, then the stored hash will no longer match the hash of the new EULA and you know to present the EULA.

**EulaSettings.swift**
{% highlight swift %}
public class EulaSettings: NSObject {
  // BundleReader retrieves the EULA html doc
  let bundleReader: BundleReader
  // AppSettings retrieves the stored EULA hash
  let appSettings: AppSettings
  
  public init(bundleReader: BundleReader, appSettings: AppSettings) {
    self.bundleReader = bundleReader
    self.appSettings = appSettings
  }
  
  func shouldPresent() -> Bool {
    return !hasAccepted()
  }
  
  func acceptEula() {
    appSettings.eulaHash = bundleReader.contentsHash
  }
  
  private func hasAccepted() -> Bool {
    
    guard let storedHash = appSettings.eulaHash where storedHash.length > 0,
      let contentsHash = bundleReader.contentsHash else {
      return false
    }
    
    return storedHash == contentsHash
  }
}
{% endhighlight %}


**EulaBundleReader.swift**
{% highlight swift %}
public protocol BundleReader {
  var filePath: String? { get }
  var contents: NSData? { get }
  var contentsHash: String? { get }
}

public class EulaBundleReader: NSObject, BundleReader {
  private(set) public var filePath: String?
  private(set) public var contents: NSData?
  private(set) public var contentsHash: String?
  
  init(bundle: NSBundle) {
    super.init()
    if let path = bundle.pathForResource("eulatext", ofType: "html") {
      filePath = path
      loadContents()
    }
  }
  
  private func loadContents() -> Void {
    guard let unwrappedContents = NSData(contentsOfFile: filePath!) else {
      return
    }
    contents = unwrappedContents
    contentsHash = unwrappedContents.computeHash()
  }
}
{% endhighlight %}

**NSDataExtensions.swift**
{% highlight swift %}
extension NSData {
  
  func computeHash() -> String {
    var digest = [UInt8](count:Int(CC_SHA1_DIGEST_LENGTH), repeatedValue: 0)
    CC_SHA1(bytes, CC_LONG(length), &digest)
    return digest.map { String(format: "%02x", $0) }.reduce("") {
       $0 + $1
    }
  }
}
{% endhighlight %}

**AppSettings.swift**
{% highlight swift %}
public class AppSettings: NSObject {
  enum Key: String {
    case EulaHash = "eulahash"
    ...
  }
  
  private let userDefaults: NSUserDefaults
  
  override init() {
    userDefaults = NSUserDefaults.standardUserDefaults()
    super.init()
  }
  
  var eulaHash: String? {
    get {
      return userDefaults.stringForKey(Key.EulaHash.rawValue)
    }
    set {
      if let value = newValue {
        userDefaults.setValue(value, forKey: Key.EulaHash.rawValue)
      } else {
        userDefaults.removeObjectForKey(Key.EulaHash.rawValue)
      }
    }
  }
}
{% endhighlight %}