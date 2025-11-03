The `qt5` formula resides [here, in Github](https://github.com/Homebrew/homebrew-core/blob/master/Aliases/qt5) and by backtracking to a previous commit you can find a previous version of the formula. The 5.6.1-1 version can be found [here](https://github.com/Homebrew/homebrew-core/blob/fdfc724dd532345f5c6cdf47dc43e99654e6a5fd/Formula/qt5.rb).

So to install Qt 5.6.1-1 with Homebrew you can do this:

```
curl -O https://raw.githubusercontent.com/Homebrew/homebrew-core/fdfc724dd532345f5c6cdf47dc43e99654e6a5fd/Formula/qt5.rb
brew install ./qt5.rb
```