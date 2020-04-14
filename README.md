# NestedFolders
Nested Folders (a bit hacky)

## Table of Contents
1. The Methods that worked before
2. The Hooks
3. The Hack
4. The Patches
 4.1 The blurred background
 4.2 The status bar
5. Future ideas
6. Final Notes

## 1. The Methods that worked before

Previously (up until iOS 11), nested folders was just a developer setting disabled for release builds on iOS.
They could easily be enabled with a few hooks that are still required today. But there is something that prevents Folders from staying in folders: The Icon Model.

## 2. The Hooks

Here are the basic methods to override in order to even drop Folders or create Folders inside folders:

```objc
%hook SBFolderController

-(BOOL)shouldOpenFolderIcon:(SBIcon *)icon {
  return TRUE;
}

-(BOOL)canAcceptFolderIconDrags {
  return TRUE;
}

%end

%hook SBIconView

-(BOOL)canReceiveGrabbedIcon:(id)icon {
  return TRUE;
}

%end

%hook SBHIconManager

- (BOOL)allowsNestedFolders {
  return TRUE;
}

// Probably redundant, due to SBIconView canReceiveGrabbedIcon
- (BOOL)icon:(SBIconView *)receiver canReceiveGrabbedIcon:(SBIconView *)received {
  return TRUE;
}

%end

%hook SBHFolderSettings

-(BOOL)allowNestedFolders {
	return TRUE;
}

-(void)setAllowNestedFolders:(BOOL)allow {
  %orig(TRUE);
}

%end
```

## 3. The Hack

Those hooks allow you to put folders inside folders. The issue is, that they don't stay there. They get dropped out.
I then found a method in `SBHIconModel` called `-(void)checkModelConsistencyInRootFolder:(id)arg1;`. Overriding this method and preventing the original implementation allows Folders to stay inside folders! They are not getting kicked out.
The problem arises is, that the Icon State is not consistent anymore, which means that after a respring, the changes would sometimes be reverted. This function seems to play a vital role in the icon state.
Digging into this function to see what it does any why it causes the problems, I noticed that it uses the `SBRootFolderWithDock` `- (NSSet *)folderIcons`. So whatever it does with kicking out folder icons, it is doing so by getting all folder icons. I applied the following hack:

```objc
static BOOL CheckingModelConsistency = NO;

%hook SBHIconModel

-(void)checkModelConsistencyInRootFolder:(id)rootFolder {
	CheckingModelConsistency = TRUE;
	%orig(rootFolder);
	CheckingModelConsistency = FALSE;
}

%end

%hook SBRootFolderWithDock

- (id)folderIcons {
	if (CheckingModelConsistency) {
		return [NSSet set];
	}
	return %orig;
}

%end
```

The idea is, when `SBHIconModel` is checking for the model consistency, we pretend there are no folder icons. So the model is actually checked and all icons are checked, just not the folders.

## 4. The Patches

Now that we can create infinite folders inside folders, there are two UI bugs we needed to fix.

### 4.1 The blurred background

Every time you open a folder it would stack another layer of blur which at some point would render a dark background which doesn't look nice. What I did was I hid the background of all folders on level 2 or deeper (level 0 is no folder opened, level 1 is one folder opened).

```objc
%hook SBFolderControllerBackgroundView

-(void)layoutSubviews {
	%orig;

	id delegate = self.delegate;
	if (delegate && [delegate isKindOfClass:%c(SBFolderController)]) {
		SBFolderController *folderController = (SBFolderController *)delegate;
		if (![folderController isKindOfClass:%c(SBRootFolderController)]) {
			self.alpha = 0.0f;
		}
	}
}

%end
```

We hide every background view not related to the first opened folder.

### 4.2 The status bar

For whatever reason starting on level 2, SpringBoard decides to show the status bar again and then for every folder stacks another layer of status bar on the screen. Probably because it thinks the first folder's view is disappearing so it shows or unhides the status bar.

```objc
%hook SBFolderController

%new 
- (BOOL)isNestedFolderController {
  SBIconController *iconController = [%c(SBIconController) sharedInstance];
	if (iconController.rootFolderController) {
	  return (iconController.rootFolderController.innerFolderController != self);
	}
	return FALSE;
}

-(id)fakeStatusBarForFolderController:(SBFolderController *)folderController {
	if ([self isNestedFolderController]) {
	  return NULL;
	}
	return %orig;
}

%end
```

This fixes the status bar for level 2 or deeper.

## 5. Future ideas

This approach is currently very tricky as we are hacking around with the icon model instead of making folders work natively inside folders.
It seems that our glue is holding the folders inside folders and not iOS itself. It works well however.

Future ideas may include faking the nested folders icon location to Dock, since Folders inside Dock are allowed. We'd need to detect if a Folder is inside a folder already. Currently a folder is still considered to be on the root page, instead of in a folder - even though it is in a folder.

# 6. Final notes

This feature is fully implemented in Springtomize 5, which is a paid package according to this document. Feel free to use the knowledge gained from this document in your Tweaks, however please link this document and give credit. You are also very welcome to improve this with any information you have.
