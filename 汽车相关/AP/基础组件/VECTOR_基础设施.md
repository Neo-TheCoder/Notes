# `amsr::generic::Singleton`
## 类成员变量
注意到，`Singleton`类竟然也在维护`引用计数`
```cpp
template <typename T>
class Singleton {
 public:
// ...

private:
  /*!
   * \brief The list of current state of static instance.
   */
  enum class InitState : std::uint8_t { uninitialized, changing, initialized };

  /*!
   * \brief   A structure to define current state, optional and reference counter.
   * \details Static optional is used to prevent dependency to other static storage variables,
   */
  struct Data {
    /*! \brief The initial state of static instance is assigned as uninitialized */
    std::atomic<InitState> init_state_{InitState::uninitialized};

    /*! \brief The container to store the data of ref_counter_ */
    ara::core::Optional<T> opt_;

    /*! \brief Reference counter for static instance */
    internal::RefCounter ref_counter_;
  };

  /*!
   * \brief The contained data for init_state_, opt_ and ref_counter_.
   */
  Data data_;
};
```


## 接口
```cpp
  /*!
   * \brief      Create a static instance and set current state to initialized.
   * \details    Emplace a value if the state is unintialized, otherwise abort.
   * \tparam     Args The types of arguments given to this function.
   * \param      args Arguments to be emplaced into the data.
   * \pre        State is InitState::uninitialized.
   * \threadsafe TRUE
   * \vprivate Product private
   */
  template <typename... Args>
  void Create(Args&&... args) noexcept {
    InitState const status{data_.init_state_.exchange(InitState::changing)};
    if (status != InitState::uninitialized) {
      ara::core::Abort("amsr::generic::Singleton::Create(): Concurrent init state change!");
    }
    data_.opt_.emplace(std::forward<Args>(args)...);
    data_.init_state_ = InitState::initialized;
  }
```

`data_.opt_.reset();`调用到真正类型的析构函数
```cpp
  /*!
   * \brief      Destroy a static instance and reset current state to uninitialized.
   * \details    Abort when the state is uninitialized or when reference counter is still referenced.
   * \pre        State is InitState::initialized and not used anymore (SingletonAccessGetRefCount() == 0).
   * \threadsafe TRUE
   * \vprivate Product private
   */
  void Destroy() {
    InitState const status{data_.init_state_.exchange(InitState::changing)};
    if (status != InitState::initialized) {
      ara::core::Abort("amsr::generic::Singleton::Destroy(): Concurrent init state change!");
    }
    if (data_.ref_counter_.HasReferences()) {
      ara::core::Abort("amsr::generic::Singleton::Destroy(): Still referenced!");
    }
    data_.opt_.reset();
    data_.init_state_ = InitState::uninitialized;
  }
```














