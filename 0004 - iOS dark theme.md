# iOS dark theme

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0009.jpeg)

> This is the article created at Jan 21, 2018 and moved from Medium.

I like applications that have implemented at least two color themes: light and dark. Especially now when the best mobile phones have OLED screens which can show perfect black color.
<!--more-->

I’m creating simple application for keeping track of cryptocurrency prices and I decided to support two themes. After implementing all stuff my application looks like on following images.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0010.png)

And two more (with charts)…

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0011.png)

I will describe shortly how I implemented that feature.

#### Storing chosen theme

In our application we have to store information about style chosen by the user. Of course the best solution is to choose `CoreData`. We have to add a new entity e.g. `Settings` and add one attribute `isDarkMode` (`Boolen` type).

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0012.png)

Obviously if we plan to support more then two themes we have to choose other data type (for example you can choose`Integer` type and create enum type whith all themes).

Also I’ve created one simple helper class for retrieve/save settings.

```swift
import Foundation
import CoreData
import UIKit

class SettingsHandler : CoreDataHandler {
    
    func getDefaultSettings() -> Settings {
        
        var settingsList:[Settings] = []
        
        let context = self.getManagedObjectContext()
        let fetchRequest = NSFetchRequest<NSFetchRequestResult>(entityName: "Settings")
        do {
            settingsList = try context.fetch(fetchRequest) as! [Settings]
        }
        catch {
            print("Error during fetching favourites")
        }
        
        var settings:Settings? = nil
        if settingsList.count == 0 {
            settings = self.createSettingsEntity()
            self.save()
        }
        else {
            settings = settingsList.first
        }
        
        return settings!
        
    }
    
    func save(settings: Settings) {
        self.save()
    }
    
    private func createSettingsEntity() -> Settings
    {
        let context = self.getManagedObjectContext()
        return Settings(context: context)
    }
}
```

Now when user chooses style we have to store that information in our application. When application starts we have to read that information and prepare proper style.

#### Notifications

In our application we need to have place where user can choose style. When user changes the style we must send a signal to all our running views. For that purposes we will use notifications. First we have to prepare two custom notifications.

```swift
import Foundation
extension Notification.Name {
    static let darkModeEnabled = Notification.Name("net.sltch.vcoin.notifications.darkModeEnabled")
    static let darkModeDisabled = Notification.Name("net.sltch.vcoin.notifications.darkModeDisabled")
}
```

Now we should create action in our application where user can switch between themes. In my application I have that place in Settings view:

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0013.png)

I have one action connected to `Dark mode` switch.

```swift
@IBAction func toggleDarkModeSwitch(_ sender: UISwitch) {
    self.settings?.isDarkMode = sender.isOn
    self.settingsHandler.save(settings: self.settings)
    
    NotificationCenter.default.post(name: sender.isOn ? .darkModeEnabled : .darkModeDisabled, object: nil)
}
```

Now when user changes theme we can catch that notification everywhere in the application.

#### Base controllers

Of course, it would not be optimal if we caught the notification separately in each view. We can create base controllers classes and customize most of the things here. When we have soemthing specific in our view we can override one of method: `enableDarkMode` or `disableDarkMode`.

Base controller for table views.

```swift
import Foundation
import UIKit

class BaseTableViewController : UITableViewController {

    var settingsHandler = SettingsHandler()
    var settings:Settings!
    
    // MARK: - View loading
    
    override func viewDidLoad() {
        super.viewDidLoad()
        self.clearsSelectionOnViewWillAppear = false
        
        NotificationCenter.default.addObserver(self, selector: #selector(darkModeEnabled(_:)), name: .darkModeEnabled, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(darkModeDisabled(_:)), name: .darkModeDisabled, object: nil)
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        
        self.settings = self.settingsHandler.getDefaultSettings()
        self.settings.isDarkMode ? enableDarkMode() : disableDarkMode()
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self, name: .darkModeEnabled, object: nil)
        NotificationCenter.default.removeObserver(self, name: .darkModeDisabled, object: nil)
    }
        
    // MARK: - Theme
    
    @objc func darkModeEnabled(_ notification: Notification) {
        enableDarkMode()
        self.tableView.reloadData()
    }
    
    @objc func darkModeDisabled(_ notification: Notification) {
        disableDarkMode()
        self.tableView.reloadData()
    }
    
    open func enableDarkMode() {
        self.view.backgroundColor = UIColor.black
        self.tableView.backgroundColor = UIColor.black
        self.navigationController?.navigationBar.barStyle = .black
        self.navigationController?.view.backgroundColor = UIColor.black
    }
    
    open func disableDarkMode() {
        self.view.backgroundColor = UIColor.white
        self.tableView.backgroundColor = UIColor.white
        self.navigationController?.navigationBar.barStyle = .default
        self.navigationController?.view.backgroundColor = UIColor.white
    }
    
    // MARK: - Table view data source
    
    override func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        
        if self.settings.isDarkMode {
            cell.textLabel?.textColor = UIColor.white
            cell.detailTextLabel?.textColor = UIColor.white
            cell.backgroundColor = UIColor.black
            
            cell.setSelectedColor(color: UIColor.darkBackground)
        }
        else {
            cell.textLabel?.textColor = UIColor.black
            cell.detailTextLabel?.textColor = UIColor.black
            cell.backgroundColor = UIColor.white
            
            cell.setSelectedColor(color: UIColor.lightBackground)
        }
    }
}
```

Base controller for view.

```swift
import Foundation
import UIKit

class BaseViewController: UIViewController {
    
    var settingsHandler = SettingsHandler()
    var settings:Settings!
    
    // MARK: - View loading
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        NotificationCenter.default.addObserver(self, selector: #selector(darkModeEnabled(_:)), name: .darkModeEnabled, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(darkModeDisabled(_:)), name: .darkModeDisabled, object: nil)
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        
        self.settings = self.settingsHandler.getDefaultSettings()
        self.settings.isDarkMode ? enableDarkMode() : disableDarkMode()
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self, name: .darkModeEnabled, object: nil)
        NotificationCenter.default.removeObserver(self, name: .darkModeDisabled, object: nil)
    }
    
    // MARK: - Theme
    
    @objc func darkModeEnabled(_ notification: Notification) {
        enableDarkMode()
    }
    
    @objc func darkModeDisabled(_ notification: Notification) {
        disableDarkMode()
    }
    
    open func enableDarkMode() {
        self.view.backgroundColor = UIColor.black
        self.navigationController?.navigationBar.barStyle = .black
    }
    
    open func disableDarkMode() {
        self.view.backgroundColor = UIColor.white
        self.navigationController?.navigationBar.barStyle = .default
    }
}
```

Now when user changes theme our notification is catched in one of our base controller and application is changing the style of views.

You can check the whole solution in GitHub repository: [mczachurski/vcoin](https://github.com/mczachurski/vcoin).

I hope this helps you!