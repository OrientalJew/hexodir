---
title: ViewPager局部刷新
date: 2018-07-27 
tags: 
- Fragment
categories:
- Android
---

<!-- toc -->


> 在开发过程中，经常会用到ViewPager与Fragment实现多页面切换效果，有时，我们想要局部刷新某些Fragment，而其他Fragment保持状态不变，该如何做到呢？
<!--more--> 
**先上代码！**

    /**
     * Created by HSH .
     */
    public abstract class BaseFragmentPagerAdapter extends FragmentPagerAdapter {
    
        private FragmentManager mFragmentManager;
    
        //保存每个Fragment的Tag，刷新页面的依据
        protected SparseArray<String> tags = new SparseArray<>();
    
        public BaseFragmentPagerAdapter(FragmentManager fm) {
            super(fm);
            mFragmentManager = fm;
        }
    
        @Override
        public Object instantiateItem(ViewGroup container, int position) {
            //得到缓存的fragment
            Fragment fragment = (Fragment) super.instantiateItem(container, position); 
            String tag = fragment.getTag();
            //保存每个Fragment的Tag
            tags.put(position, tag);
            return fragment;
        }
    
        //拿到指定位置的Fragment
        public Fragment getFragmentByPosition(int position) {
            return mFragmentManager.findFragmentByTag(tags.get(position));
        }
    
        public List<Fragment> getFragments(){
            return mFragmentManager.getFragments();
        }
    
        //刷新指定位置的Fragment
        public void notifyFragmentByPosition(int position) {
            tags.removeAt(position);
            notifyDataSetChanged();
        }
    
        @Override
        public int getItemPosition(Object object) {
            Fragment fragment = (Fragment) object;
            //如果Item对应的Tag存在，则不进行刷新
            if (tags.indexOfValue(fragment.getTag()) > -1) {
                return super.getItemPosition(object);
            }
            return POSITION_NONE;
        }
    }
    
该类是本人最近在项目中使用的，只需继承该类即可。

    /**
     * Created by HSH .
     */
    public class CustomLrcPagerAdapter extends BaseFragmentPagerAdapter {
        private List<String> lrcs = new ArrayList<>();
        private MusicInfo info;
    
        public CustomLrcPagerAdapter(FragmentManager fm, MusicInfo info) {
            super(fm);
            this.info = info;
        }
    
        public void addDatas(List<String> lrcs) {
            this.lrcs.addAll(lrcs);
            notifyDataSetChanged();
        }
    
        @Override
        public Fragment getItem(int position) {
            return CustomLrcFragment.newInstance(info, lrcs.get(position), position);
        }
    
        //除了给定位置，其他位置的Fragment进行刷新
        public void notifyChangeWithoutPosition(int position) {
            String valueP = tags.valueAt(position);
            tags.clear();
            tags.put(position, valueP);
            notifyDataSetChanged();
        }
    
    
        @Override
        public int getCount() {
            return lrcs.size();
        }
    }
 
 
刷新的核心原理很简单，相信看过源码的都会，在PagerAdapter中提供了一个方法：

    /**
     * Called when the host view is attempting to determine if an item's position
     * has changed. Returns {@link #POSITION_UNCHANGED} if the position of the given
     * item has not changed or {@link #POSITION_NONE} if the item is no longer present
     * in the adapter.
     *
     * <p>The default implementation assumes that items will never
     * change position and always returns {@link #POSITION_UNCHANGED}.
     *
     * @param object Object representing an item, previously returned by a call to
     *               {@link #instantiateItem(View, int)}.
     * @return object's new position index from [0, {@link #getCount()}),
     *         {@link #POSITION_UNCHANGED} if the object's position has not changed,
     *         or {@link #POSITION_NONE} if the item is no longer present.
     */
    public int getItemPosition(Object object) {
        return POSITION_UNCHANGED;
    }

注释中已经说明了，当我们返回了POSITION_UNCHANGED，则表示页面数据不变，不进行更新；
返回POSITION_NONE，则表示页面不存在，需要进行更新。

因此，我重写了该方法：

        @Override
        public int getItemPosition(Object object) {
            Fragment fragment = (Fragment) object;
            //如果Item对应的Tag存在，则不进行刷新
            if (tags.indexOfValue(fragment.getTag()) > -1) {
                return super.getItemPosition(object);
            }
            return POSITION_NONE;
        }


hihi，是不是很简单！

在接触公司项目的过程中（本人新来的），发现公司项目中ViewPager+Fragment组合的使用方式存在问题，估计很多人也这么用过，就是定义一个集合用来缓存放到ViewPager中的Fragment，类似我们公司项目的这种做法：

    /**
     * Created by Administrator on 2016/11/30.
     */
    public class ViewPagerAdapter extends FragmentStatePagerAdapter {
    
        private List<Fragment> mList_Fragment = new ArrayList<>();
        private HashMap<Integer, Boolean> mList_Need_Update = new HashMap<>();
        private FragmentManager mFragmentManager;
    
        public ViewPagerAdapter(FragmentManager fm, List<Fragment> fragments) {
            super(fm);
            mFragmentManager = fm;
            mList_Need_Update.clear();
            mList_Fragment.clear();
            if (fragments != null) {
                mList_Fragment.addAll(fragments);
            }
        }
    
    //    @Override
    //    public Object instantiateItem(ViewGroup container, int position) {
    //        Fragment fragment = (Fragment) super.instantiateItem(container, position); //得到缓存的fragment
    //
    //        Boolean update = mList_Need_Update.get(position);
    //        if (update != null && update) {
    //            String fragmentTag = fragment.getTag(); //得到tag，这点很重要
    //            FragmentTransaction ft = mFragmentManager.beginTransaction();
    //            ft.remove(fragment); //移除旧的fragment
    //            fragment = getItem(position); //换成新的fragment
    //            ft.add(container.getId(), fragment, fragmentTag); //添加新fragment时必须用前面获得的tag，这点很重要
    //            ft.attach(fragment);
    //            ft.commit();
    //            mList_Need_Update.put(position, false); //清除更新标记（只有重新启动的时候需要去创建新的fragment对象），防止正常情况下频繁创建对象
    //        }
    //
    //        return fragment;
    //    }
    
        public List<Fragment> getListFragment(){
            return mList_Fragment;
        }
    
        public void setListFragment(List<Fragment> list_Fragment) {
    //        if(list_Fragment != null){
    //            FragmentTransaction ft = mFragmentManager.beginTransaction();
    //            for (int i = 0; i < mList_Fragment.size(); i++) {
    //                Fragment fragment = (Fragment) mList_Fragment.get(i);
    //                ft.remove(fragment);
    //            }
    //            ft.commit();
    //            ft = null;
    //            mFragmentManager.executePendingTransactions();
    //        }
            mList_Need_Update.clear();
            this.mList_Fragment.clear();
            if (list_Fragment != null) {
                this.mList_Fragment.addAll(list_Fragment);
            }
            notifyDataSetChanged();
        }
    
        public void setListNeedUpdate(List<Fragment> fragments) {
            mList_Fragment.clear();
            if (fragments != null) {
                mList_Fragment.addAll(fragments);
            }
            mList_Need_Update.clear();
            for (int i = 0; i < mList_Fragment.size(); i++) {
                mList_Need_Update.put(i, true);
            }
        }
    
        @Override
        public Fragment getItem(int position) {
            if(mList_Fragment.size() < position){
                return null;
            }
            return mList_Fragment.get(position);
        }
    
    
        @Override
        public int getCount() {
            return mList_Fragment.size();
        }
    
        @Override
        public int getItemPosition(Object object) {
            return PagerAdapter.POSITION_NONE;
        }
    
        @Override
        public void restoreState(Parcelable state, ClassLoader loader) {
            try {
                super.restoreState(state, loader);
            } catch (Exception e) {
    
            }
    
        }
    }


>这种做法看似方便我们操作ViewPager中的Fragment，但是存在一个很致命的问题。

某些情况下，我们在从其他页面回退到ViewPager，在进行Fragment数据更新时，会发现居然没有效果（或者效果很诡异，例如会出现多次调用的情况）。

这种情况其实就是Fragment进行了热启动。（我的说法不知是否准确，指的就是内存不足时，页面被销毁了并调用了onSaveInstanceState方法，在重新回到页面时，我们可以从Bundle savedInstanceState中拿到之前缓存的数据。）

由于FragmentPagerAdapter中的FragmentManager已经帮我们缓存了所有Fragment，并且在数据恢复时，也自动帮我们进行恢复处理。
所以，个人猜测 **（未经源码验证的！）** ，在FragmentManager进行数据恢复时，如果我们本地通过集合缓存了一份Fragment，则这份Fragment与FragmentManager进行数据恢复后的Fragment是不同的！

我个人的做法是，每次需要操作ViewPager中的Fragment时，都从FragmentManager中拿：

        //拿到指定位置的Fragment
        public Fragment getFragmentByPosition(int position) {
            return mFragmentManager.findFragmentByTag(tags.get(position));
        }
    
        public List<Fragment> getFragments(){
            return mFragmentManager.getFragments();
        }


东西很简单，但相信对新手还是有点用处的，至少本人当初在这个问题上是被整得欲仙欲死  /(ㄒoㄒ)/~~

_说明一下，以上只是个人猜测，本人懒，没去看源码，如果有哪位看官知道原理的，或者本人理解存在错误的，还请在评论中进行指正！谢谢！_
