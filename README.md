# 2017
## 2/7
#####1,移除设置中的内核版本
	packages\apps\Settings\src\com\android\settings\DeviceInfoSettings.java
   
    	public void onCreate(Bundle icicle) {
        ...   
     [+] getPreferenceScreen().removePreference(findPreference(KEY_KERNEL_VERSION));
        
#####2，移除IP拨号
	packages/services/Telephony/src/com/android/phone/CallFeaturesSetting.java 
    	
        private void setIpFunction() {
        ...
         if (prefIp != null) {
          [-]  prefIp.setIntent(intent);
          [+] getPreferenceScreen().removePreference(prefIp);
          }
                  
packages/apps/Dialer/src/com/android/dialer/dialpad/DialpadFragment.java 
		
        private PopupMenu buildOptionsMenu(View invoker) {
        ... ...
        
         [-]/*menu.findItem(R.id.menu_ip_dial).setVisible(
                     DialerFeatureOptions.IP_PREFIX && enable
                     && !PhoneNumberHelper.isUriNumber(mDigits.getText().toString()));*/
         [+] menu.findItem(R.id.menu_ip_dial).setVisible(false);
        
packages/apps/Dialer/src/com/android/dialer/calllog/CallLogListItemViewHolder.java 
       
       private void bindActionButtons() {
       ... ...
        /** M: [IP Dial] Add IP Dial @{ */
        if (DialerFeatureOptions.IP_PREFIX && canPlaceCallToNumber
                && !PhoneNumberHelper.isUriNumber(number)) {
            /// M: [Suggested Account] Supporting suggested account @{
            if (DialerFeatureOptions.isSuggestedAccountSupport()) {
                ipDialButtonView.setTag(IntentProvider
                        .getSuggestedIpDialCallIntentProvider(number, accountHandle));
            } else {
                ipDialButtonView.setTag(IntentProvider
                        .getIpDialCallIntentProvider(number));
            }
            /// @}
            [-]ipDialButtonView.setVisibility(View.VISIBLE);
            [+]ipDialButtonView.setVisibility(View.GONE);
        } else {
            ipDialButtonView.setVisibility(View.GONE);
        }


#####3,拨号匹配数为9
vendor/mediatek/proprietary/frameworks/base/packages/FwkPlugin/src/com/mediatek/op/telephony/PhoneNumberExt.java 
        
        
		public int getMinMatch() {
       	[-]return 7;
       	[+]return 9;
        }

























