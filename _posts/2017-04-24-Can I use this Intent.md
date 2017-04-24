# Can I use this Intent ?

#### Android 提供了强大而有用的(intents), 它不仅可以应用在应用内部，还能用于不同应用之间</br>

<br><br><br>




### 在使用它时可能会出现**ActivityNotFoundException**,这是因为未对系统中是否存在相应的intent接受者做判断. 可以使用以下方法增强代码健壮性:<br>
    /**
     * Indicates whether the specified action can be used as an intent. This
     * method queries the package manager for installed packages that can
     * respond to an intent with the specified action. If no suitable package is
     * found, this method returns false.
     *
     * @param context The application's environment.
     * @param action The Intent action to check for availability.
     *
     * @return True if an Intent with the specified action can be sent and
     *         responded to, false otherwise.
     */
    public static boolean isIntentAvailable(Context context, String action) {
        final PackageManager packageManager = context.getPackageManager();
        final Intent intent = new Intent(action);
        List<ResolveInfo> list =
                packageManager.queryIntentActivities(intent,
                        PackageManager.MATCH_DEFAULT_ONLY);
        return list.size() > 0;
    }

Here is how I use it??<br>
<pre><code>
           @Override
           public boolean onPrepareOptionsMenu(Menu menu) {
               final boolean scanAvailable = isIntentAvailable(this,
                   "com.google.zxing.client.android.SCAN");

               MenuItem item;
               item = menu.findItem(R.id.menu_item_add);
               item.setEnabled(scanAvailable);

               return super.onPrepareOptionsMenu(menu);
           }
</code></pre>

