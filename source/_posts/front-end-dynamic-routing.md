---
title: 前端动态路由
date: 2024-01-20 11:05:01
tags:
    - 前端
    - 动态路由
    - React
    - Antd
    - Umi
categories: 前端-React
---

# 为什么需要动态路由
在管理系统中，不应该将所有的权限开放给所有的用户，而是需要对访问权限进行控制，从而保证数据的安全。

给不同的角色分配不同的权限，可以保护敏感信息、保障操作的专业性、提高系统的安全。比如：财务信息只能由财务操作，员工信息交给人事管理。

使用动态路由就是为了根据不同的用户，展示不同的系统页面，杜绝掉一些不合法的数据操作。

# 方案比较

## 前端控制
前端使用routes + access控制用户的访问页面。

## 后端返回
调用后端的接口，然后将返回的页面配置渲染出来。

## 前后端结合
前端使用配置routes，用户登录后调用后端的接口。根据后端返回的数据与前端routes配置做对比，将一致的渲染出来。

# 实现(基于Ant Design Pro)

## 前端控制
TODO
## 后端返回
TODO

## 前后端结合
1. 首先要配置`config`中的`routes`
2. 在`src/app.ts`的`getInitialState()`初始化全局状态, 提供一些全局化方法.
    ```ts
    export async function getInitialState(): Promise<{
        userMenus?: API.Menu[];
        getMenus?: => Promise<API.Menu[] | undefined>;
    }> {
        const getMenus = async (isTree: boolean) => {
            try {
                const msg = await getCurrentUserMenus();
                return msg.data?.list;
            } catch (error) {
                console.error('获取菜单失败:', error);
                history.push(LOGIN_PATH);
            }
            return undefined;
        };

        // 如果不是登录页面，执行
        const { location } = history;
        if (location.pathname !== LOGIN_PATH) {
            const userMenus = await getMenus();
            return {
                getMenus, 
                userMenus,
                settings: defaultSettings as Partial<LayoutSettings>,
            };
        }

        return {
            userMenus,
            getMenus,
            settings: defaultSettings as Partial<LayoutSettings>,
        };
    }
    ```
3. 在用户登录后调用初始化的全局方法, 设置初始值
    ```ts
    const getMenus = async () => {
        const menus = await initialState?.getMenus?.();
        if (menus) {
        flushSync(() => {
            setInitialState((s) => ({
            ...s,
            userMenus: menus,
            }));
        });
        }
    };

    const handleSubmit = async (values: API.AccountSignInReq) => {
        try {
            // 登录
            const msg = await userSignIn(values, { skipErrorHandler: true });
            if (msg.success) {
                const defaultLoginSuccessMessage = intl.formatMessage({
                    id: 'pages.login.success',
                    defaultMessage: '登录成功！',
                });
                message.success(defaultLoginSuccessMessage);
                localStorage.setItem(TOKEN_KEY, msg.data.access_token || '');

                await getMenus();

                const urlParams = new URL(window.location.href).searchParams;
                history.push(urlParams.get('redirect') || '/');
                return;
            }
        } catch (error) {
            const defaultLoginFailureMessage = intl.formatMessage({
                id: 'pages.login.failure',
                defaultMessage: '登录失败，请重试！',
            });
            message.error(defaultLoginFailureMessage);
        }
    };
    ```
4. 在`src/app.ts`的`layout`中过滤掉没有权限的路由
    ```ts
    export const layout: RunTimeLayoutConfig = ({ initialState, setInitialState }) => {
        const ignorePaths = ['/user/login', '/', '/404'];
        let userMenus = initialState?.userMenus;

        // 过滤菜单项，保留在 routes 中存在的路径
        const filterMenuItems = (menuItems: any[]) => {
            return menuItems.filter((item) => {
            if (ignorePaths.includes(item.path)) {
                return true;
            }

            if (!userMenus || userMenus.length === 0) {
                return false;
            }

            // 检查当前菜单项的路径是否在 routes 中存在
            const shouldKeep = userMenus?.some((route) => route.path === item.path);
            if (shouldKeep && item.children) {
                item.children = filterMenuItems(item.children);
            }
            // 保留当前菜单项或其子菜单中有需要保留的项
            return shouldKeep;
            });
        };

        return {
            menuDataRender: (menuItems) => filterMenuItems(menuItems),
        }
    };
    ```
5. 在页面改变时再次检查路由, 防止用户手动输入没有权限的路径
   ```ts
    export const layout: RunTimeLayoutConfig = ({ initialState, setInitialState }) => {
        return {
            onPageChange: () => {
                const { location } = history;
                const { userMenus } = initialState || {};
                // 如果用户没有当前路径的权限, 跳回到主页面
                if (userMenus && !userMenus.some((item) => item.path === location.pathname)) {
                    history.push(HOME_PATH);
                }
            },
            
        };
    };
   ```