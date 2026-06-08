package com.pro.ai.assistant;

import android.accessibilityservice.AccessibilityService;
import android.accessibilityservice.AccessibilityServiceInfo;
import androidx.localbroadcastmanager.content.LocalBroadcastManager;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.IntentFilter;
import android.content.Intent;
import android.util.Log;
import android.view.accessibility.AccessibilityEvent;
import android.view.accessibility.AccessibilityNodeInfo;
import android.widget.Toast;

import java.util.List;

public class ProAccessibilityService extends AccessibilityService {

    private BroadcastReceiver commandReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String commandType = intent.getStringExtra(AssistantService.EXTRA_COMMAND_TYPE);
            String packageName = intent.getStringExtra(AssistantService.EXTRA_PACKAGE_NAME);

            String response = "";
            if (AssistantService.COMMAND_READ_SCREEN.equals(commandType)) {
                response = readScreenContent();
            } else if (AssistantService.COMMAND_OPEN_APP.equals(commandType)) {
                if (packageName != null) {
                    openApp(packageName);
                    response = "تم فتح التطبيق.";
                } else {
                    response = "لم يتم تحديد اسم التطبيق.";
                }
            } else if (AssistantService.COMMAND_PERFORM_GLOBAL_ACTION.equals(commandType)) {
                String actionName = intent.getStringExtra(AssistantService.EXTRA_GLOBAL_ACTION_TYPE);
                if (actionName != null) {
                    performGlobalActionCommand(actionName);
                    // Response will be sent from performGlobalActionCommand
                    return;
                } else {
                    response = "لم يتم تحديد الإجراء العالمي.";
                }
            } else if (AssistantService.COMMAND_CAMERA_VISION.equals(commandType)) {
                String imagePath = intent.getStringExtra(AssistantService.EXTRA_IMAGE_PATH);
                String error = intent.getStringExtra("error");
                if (error != null) {
                    response = "حدث خطأ أثناء التقاط الصورة: " + error;
                } else if (imagePath != null) {
                    // Placeholder for actual vision model processing
                    response = "تم التقاط الصورة من المسار: " + imagePath + ". جاري تحليل الصورة (وظيفة نموذج الرؤية).";
                } else {
                    response = "لم يتم التقاط صورة.";
                }
            }
            // Send response back to AssistantService
            Intent responseIntent = new Intent(AssistantService.ACTION_ACCESSIBILITY_COMMAND);
            responseIntent.putExtra("response", response);
            LocalBroadcastManager.getInstance(context).sendBroadcast(responseIntent);
        }
    };

    private static final String TAG = "ProAccessibilityService";

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        Log.d(TAG, "onAccessibilityEvent: " + event.getEventType());
        // This method is called when an accessibility event occurs.
        // We can filter events in accessibility_service_config.xml
    }

    @Override
    public void onInterrupt() {
        Log.d(TAG, "onInterrupt");
    }

    @Override
    protected void onServiceConnected() {
        super.onServiceConnected();
        Log.d(TAG, "onServiceConnected");
        AccessibilityServiceInfo info = new AccessibilityServiceInfo();
        info.eventTypes = AccessibilityEvent.TYPES_ALL_MASK;
        info.feedbackType = AccessibilityServiceInfo.FEEDBACK_GENERIC;
        info.notificationTimeout = 100;
        info.flags = AccessibilityServiceInfo.FLAG_REPORT_VIEW_IDS | AccessibilityServiceInfo.FLAG_RETRIEVE_INTERACTIVE_WINDOWS;
        this.setServiceInfo(info);
        Toast.makeText(this, "Pro AI Accessibility Service Connected", Toast.LENGTH_SHORT).show();
        LocalBroadcastManager.getInstance(this).registerReceiver(commandReceiver, new IntentFilter(AssistantService.ACTION_ACCESSIBILITY_COMMAND));
    }

    public String readScreenContent() {
        AccessibilityNodeInfo rootNode = getRootInActiveWindow();
        if (rootNode == null) {
            return "لا يمكن الوصول إلى محتوى الشاشة.";
        }
        StringBuilder screenContent = new StringBuilder();
        traverseNode(rootNode, screenContent, 0);
        return screenContent.toString();
    }

    private void traverseNode(AccessibilityNodeInfo node, StringBuilder content, int depth) {
        if (node == null) {
            return;
        }

        // Limit depth to avoid excessively long output
        if (depth > 5) {
            return;
        }

        String prefix = "";
        for (int i = 0; i < depth; i++) {
            prefix += "  "; // Indent for better readability
        }

        if (node.getText() != null && node.getText().length() > 0) {
            content.append(prefix).append("Text: ").append(node.getText()).append("\n");
        }
        if (node.getContentDescription() != null && node.getContentDescription().length() > 0) {
            content.append(prefix).append("Description: ").append(node.getContentDescription()).append("\n");
        }
        if (node.getViewIdResourceName() != null) {
            content.append(prefix).append("ID: ").append(node.getViewIdResourceName()).append("\n");
        }
        if (node.isClickable()) {
            content.append(prefix).append("Clickable: Yes\n");
        }

        for (int i = 0; i < node.getChildCount(); i++) {
            traverseNode(node.getChild(i), content, depth + 1);
        }
    }

    public boolean performGlobalAction(int action) {
        return super.performGlobalAction(action);
    }

    public void performGlobalActionCommand(String actionName) {
        int action = -1;
        switch (actionName.toLowerCase()) {
            case "home":
                action = GLOBAL_ACTION_HOME;
                break;
            case "back":
                action = GLOBAL_ACTION_BACK;
                break;
            case "recents":
                action = GLOBAL_ACTION_RECENTS;
                break;
            case "notifications":
                action = GLOBAL_ACTION_NOTIFICATIONS;
                break;
            case "quick_settings":
                action = GLOBAL_ACTION_QUICK_SETTINGS;
                break;
            case "power_dialog":
                action = GLOBAL_ACTION_POWER_DIALOG;
                break;
            case "lock_screen":
                if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.P) {
                    action = GLOBAL_ACTION_LOCK_SCREEN;
                }
                break;
            case "take_screenshot":
                if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.P) {
                    action = GLOBAL_ACTION_TAKE_SCREENSHOT;
                }
                break;
        }

        if (action != -1) {
            boolean success = performGlobalAction(action);
            Log.d(TAG, "Performed global action " + actionName + ": " + success);
            // Send response back to AssistantService
            Intent responseIntent = new Intent(AssistantService.ACTION_ACCESSIBILITY_COMMAND);
            responseIntent.putExtra("response", "تم تنفيذ الأمر: " + actionName);
            LocalBroadcastManager.getInstance(this).sendBroadcast(responseIntent);
        } else {
            Log.e(TAG, "Unknown global action: " + actionName);
            Intent responseIntent = new Intent(AssistantService.ACTION_ACCESSIBILITY_COMMAND);
            responseIntent.putExtra("response", "لا يمكنني تنفيذ هذا الإجراء: " + actionName);
            LocalBroadcastManager.getInstance(this).sendBroadcast(responseIntent);
        }
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        LocalBroadcastManager.getInstance(this).unregisterReceiver(commandReceiver);
    }

    public void openApp(String packageName) {
        Intent launchIntent = getPackageManager().getLaunchIntentForPackage(packageName);
        if (launchIntent != null) {
            launchIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            startActivity(launchIntent);
        } else {
            Log.e(TAG, "Could not find package: " + packageName);
        }
    }

    // Method to find and click a specific view (e.g., a button with specific text)
    public boolean findAndClick(String textToFind) {
        AccessibilityNodeInfo rootNode = getRootInActiveWindow();
        if (rootNode == null) {
            return false;
        }
        List<AccessibilityNodeInfo> nodes = rootNode.findAccessibilityNodeInfosByText(textToFind);
        for (AccessibilityNodeInfo node : nodes) {
            if (node.isClickable() && node.isVisibleToUser()) {
                node.performAction(AccessibilityNodeInfo.ACTION_CLICK);
                return true;
            }
        }
        return false;
    }

    // Example of how to interact with the service from AssistantService
    // You would typically bind to this service or use a broadcast receiver
    // For simplicity, we'll assume a direct call for now, but a proper IPC mechanism is needed.
    // This service needs to be enabled by the user in Accessibility settings.
}
