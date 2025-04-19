+++
+++

目前可以成功在本地运行FC自带的pytest测试. 所需要的patch如[`fc-local-tests.diff`](fc-local-tests.diff). 最终的测试结果如[`fc-local-tests.log`](fc-local-tests.log)

3-3 update:

目前已经可以正常从pytest中通过API调用FaaSnap的函数.

实现方式是:

1. 将API daemon以及相关函数embed到vm的rootfs.
2. API data放在测试环境的docker image中, 并在运行pytest时通过redis暴露给vm.
3. 因为redis和vm运行在同一个netns (pytest看不到), 所以额外给pytest加入了通过subprocess在给定netns中运行requests库调用API的能力
