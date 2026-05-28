```cpp
namespace sim::bin
{
  class AutopilotModelQml : public QObject
  {
    Q_OBJECT

   public:
    using AutopilotModel = sim::AutopilotModel;
    /* Or
    enum class AutopilotModel
    {
      VT45 = 0,
      VT30 = 1,
      VT440 = 2
    };*/
    Q_ENUM(AutopilotModel);
  };
}

Q_DECLARE_METATYPE(sim::bin::AutopilotModelQml::AutopilotModel)

...

qmlRegisterUncreatableType<sim::bin::AutopilotModelQml>(
      "simulatorstandalone",
      1,
      0,
      "AutopilotModel",
      "AutopilotModel is a enum class holder");
```
