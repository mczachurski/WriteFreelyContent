# Gesture recognizer for changing theme

> This is the article created at Jan 21, 2018 and moved from Medium.

In my previous article I described how I implemented changing themes between light and dark. I would like to add another feature, shortcut to change that theme. User can swipe with two fingers up or down to enable/disable dark theme (very simliar feature exists in Tweetbot — which is my favourite Twitter client).
<!--more-->

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0014.gif)

First we have to create custom `UIGestureRecognizer`. We have to recognize move of two fingers. In our recognizer we have to add function which is executed when touches have began. It is really simple function.

```swift
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent) {
        super.touchesBegan(touches, with: event)
        
        if touches.count >= 1 {
            state = .began
            return
        }
        
        state = .failed
    }
```

Now we have to implement function which is executed when user moves his fingers.

```swift
    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent) {
        super.touchesMoved(touches, with: event)
        
        if state == .failed || touches.count != 2 {
            return
        }
        
        let window = view?.window
        let arrayTouches = Array(touches)
        
        // First finger
        if let loc = arrayTouches.first?.location(in: window) {
            firstTouchedPoints.append(loc)
            state = .changed
        }

        // Second finger
        if let loc = arrayTouches.last?.location(in: window) {
            secondTouchedPoints.append(loc)
            state = .changed
        }
    }
```

In that function we are adding information about touches to two collections (only when user touches screen with two fingers).

Next function which we have to implement is function fired when user finishes his touch. In that function we have to check if user did something which we expect.

```swift
    override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent) {
        super.touchesEnded(touches, with: event)
        
        if self.twoFingersMoveUp() {
            self.fingersDirection = TwoFingersMove.moveUp
            state = .ended
            return
        }
        
        if self.twoFingersMoveDown() {
            self.fingersDirection = TwoFingersMove.moveDown
            state = .ended
            return
        }
        
        state = .failed
    }
```

As you can see we have two functions here: `twoFingersMoveUp` and `twoFingersMoveDown`. In those functions we are verifying move of both fingers and if we recognize move then we finish our gesture recognizer with state: `ended`. If we didn’t recognize move we have to finish with state: `failed`.

My functions which verifiy movement are really simple (maybe a little bit to simple :-)).

```swift
    private func twoFingersMoveUp() -> Bool {
        var firstFingerWasMoveUp = false
        if firstTouchedPoints.count > 1 && firstTouchedPoints[0].y > firstTouchedPoints[firstTouchedPoints.count - 1].y {
            firstFingerWasMoveUp = true
        }
        
        var secondFingerWasMoveUp = false
        if secondTouchedPoints.count > 1 && secondTouchedPoints[0].y > secondTouchedPoints[secondTouchedPoints.count - 1].y {
            secondFingerWasMoveUp = true
        }
        
        return firstFingerWasMoveUp && secondFingerWasMoveUp
    }
    
    private func twoFingersMoveDown() -> Bool {
        var firstFingerWasMoveDown = false
        if firstTouchedPoints.count > 1 &&  firstTouchedPoints[0].y < firstTouchedPoints[firstTouchedPoints.count - 1].y {
            firstFingerWasMoveDown = true
        }
        
        var secondFingerWasMoveDown = false
        if secondTouchedPoints.count > 1 && secondTouchedPoints[0].y < secondTouchedPoints[secondTouchedPoints.count - 1].y {
            secondFingerWasMoveDown = true
        }
        
        return firstFingerWasMoveDown && secondFingerWasMoveDown
    }
```

The last function that we have to implement is `reset` function. That function is fired between user touches. We have to clean up all the stuff which we collect during user touch.

```swift
    override func reset() {
        super.reset()
        
        self.fingersDirection = TwoFingersMove.unknown
        self.firstTouchedPoints.removeAll(keepingCapacity: true)
        self.secondTouchedPoints.removeAll(keepingCapacity: true)
        
        state = .possible
    }
```

Now we have everything in our gesture recognizer. We can connect our gesture recognizer with our controllers. For that purpose we have to add following code in our `viewDidLoad` function:

```swift
let twoFingersGestureReognizer = TwoFingersGestureRecognizer(target: self, action: #selector(twoFingersGestureRecognizer))
twoFingersGestureReognizer.cancelsTouchesInView = false
twoFingersGestureReognizer.delegate = self
self.view.addGestureRecognizer(twoFingersGestureReognizer)
```

Now our gesture recognizer will fire function `twoFingersGestureRecognizer` when touches ended. In that function we can do whatever we want. In my case I will change the theme (to light when user moves fingers up or to dark when user moves fingers down).

```swift
@objc func twoFingersGestureRecognizer(sender: TwoFingersGestureRecognizer) {
    if sender.state == .ended {
        
        if sender.fingersDirection == .moveDown && !self.settings.isDarkMode {
            self.settings?.isDarkMode = true
            self.settingsHandler.save(settings: self.settings)
            
            player.play(name: "switch-on")
            NotificationCenter.default.post(name: .darkModeEnabled, object: nil)
        }
        
        if sender.fingersDirection == .moveUp && self.settings.isDarkMode {
            self.settings?.isDarkMode = false
            self.settingsHandler.save(settings: self.settings)
            
            player.play(name: "switch-off")
            NotificationCenter.default.post(name: .darkModeDisabled, object: nil)
        }
    }
}
```

There is one more thing that we have to do. Unfortunatelly now our gesture recognizer consumes all user gestures so for example table view will not react for user touches. That’s why we have to add to our controller `UIGestureRecognizerDelegate` and implement one function (fortunately the last one):

```swift
func gestureRecognizer(_ gestureRecognizer: UIGestureRecognizer,
                       shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer) -> Bool {
    return true
}
```

Of course if we want to have that behaviour common for all screens we have to put that code in one common controller. You can take a look into that here in one of my iOS project on GitHub: [mczachurski/vcoin](https://github.com/mczachurski/vcoin).