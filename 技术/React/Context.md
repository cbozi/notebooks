Context

## Legacy Context有什么问题 

If the parent component put an updated value by that name into context, and there was an intervening component that skipped rendering using shouldComponentUpdate -> false, the nested component would never see the updated value. That's one of the reasons why the React team always discouraged people from using legacy context directly, suggesting that any uses should be wrapped up in an abstraction so that when a "future replacement for context" came out, it would be easier to migrate. (See the React "Legacy Context" docs page and Michel Westrate's article How to Safely Use React Context for more details.)
> [Idiomatic Redux: The History and Implementation of React-Redux · Mark's Dev Blog](https://blog.isquaredsoftware.com/2018/11/react-redux-history-implementation/)
