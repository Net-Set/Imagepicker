# Imagepicker



#live location code 
package com.github.dhaval2404.imagepicker.sample

import android.Manifest
import android.app.Notification
import android.app.PendingIntent
import android.app.Service
import android.content.Intent
import android.content.pm.PackageManager
import android.location.Location
import android.os.Binder
import android.os.Environment
import android.os.Handler
import android.os.IBinder
import android.os.Looper
import android.widget.Toast
import androidx.core.app.ActivityCompat
import androidx.core.app.NotificationCompat
import com.google.android.gms.location.FusedLocationProviderClient
import com.google.android.gms.location.LocationCallback
import com.google.android.gms.location.LocationRequest
import com.google.android.gms.location.LocationResult
import com.google.android.gms.location.LocationServices
import java.io.File
import java.io.FileWriter
import java.io.IOException

class LocationForegroundService : Service() {

    private val NOTIFICATION_ID = 123
    private val NOTIFICATION_CHANNEL_ID = "location_service_channel"
    private val LOCATION_UPDATE_INTERVAL = 2000 // in milliseconds

    private lateinit var fusedLocationClient: FusedLocationProviderClient
    private lateinit var locationCallback: LocationCallback
    private var isServiceRunning = false

    private lateinit var handler: Handler
    private lateinit var toastRunnable: Runnable

    override fun onCreate() {
        super.onCreate()
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
        createLocationCallback()
        handler = Handler(Looper.getMainLooper())
        setupToastRunnable()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startLocationUpdates()
        startForeground(NOTIFICATION_ID, createNotification())
        isServiceRunning = true

        // Schedule the toast task
        handler.postDelayed(toastRunnable, 2 * 60 * 1000) // 2 minutes

        return START_STICKY
    }

    override fun onDestroy() {
        stopLocationUpdates()
        stopForeground(true)
        isServiceRunning = false

        // Remove the callback
        handler.removeCallbacks(toastRunnable)

        super.onDestroy()
    }

    override fun onBind(intent: Intent?): IBinder? {
        return LocationServiceBinder()
    }

    private fun createLocationCallback() {
        locationCallback = object : LocationCallback() {
            override fun onLocationResult(locationResult: LocationResult) {
                val location = locationResult.lastLocation
                // Process the received location data
                updateNotificationWithLocation(location.latitude, location.longitude)
                // Save the location data to CSV file
                saveLocationToCsv(location.latitude, location.longitude)
            }
        }
    }

    private fun startLocationUpdates() {
        val locationRequest = LocationRequest()
            .setInterval(LOCATION_UPDATE_INTERVAL.toLong())
            .setPriority(LocationRequest.PRIORITY_HIGH_ACCURACY)
        if (ActivityCompat.checkSelfPermission(
                this,
                Manifest.permission.ACCESS_FINE_LOCATION
            ) != PackageManager.PERMISSION_GRANTED && ActivityCompat.checkSelfPermission(
                this,
                Manifest.permission.ACCESS_COARSE_LOCATION
            ) != PackageManager.PERMISSION_GRANTED
        ) {
            // TODO: Consider calling
            //    ActivityCompat#requestPermissions
            // here to request the missing permissions, and then overriding
            //   public void onRequestPermissionsResult(int requestCode, String[] permissions,
            //                                          int[] grantResults)
            // to handle the case where the user grants the permission. See the documentation
            // for ActivityCompat#requestPermissions for more details.
            return
        }
        fusedLocationClient.requestLocationUpdates(locationRequest, locationCallback, null)
    }

    private fun stopLocationUpdates() {
        fusedLocationClient.removeLocationUpdates(locationCallback)
    }

    private fun createNotification(): Notification {
        val notificationIntent = Intent(this, MainActivity::class.java)
        val pendingIntent = PendingIntent.getActivity(
            this, 0, notificationIntent, 0
        )

        return NotificationCompat.Builder(this, NOTIFICATION_CHANNEL_ID)
            .setContentTitle("Location Service")
            .setContentText("Getting live location updates")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentIntent(pendingIntent)
            .build()
    }

    private fun updateNotificationWithLocation(latitude: Double, longitude: Double) {
        // Update the notification text with new location coordinates
        // ...
    }

    private fun saveLocationToCsv(latitude: Double, longitude: Double) {
        val csvData = "$latitude,$longitude\n"

        val directory = File(
            Environment.getExternalStorageDirectory(), "LocationData"
        )
        if (!directory.exists()) {
            directory.mkdirs()
        }

        val fileName = "location_data.csv"
        val csvFile = File(directory, fileName)

        try {
            val writer = FileWriter(csvFile, true) // 'true' for append mode
            writer.append(csvData)
            writer.flush()
            writer.close()
        } catch (e: IOException) {
            e.printStackTrace()
        }
    }

    private fun setupToastRunnable() {
        toastRunnable = Runnable {
            showToastWithLocation()
            handler.postDelayed(this.toastRunnable, 2 * 60 * 1000) // Repeat every 2 minutes
        }
    }

    private fun showToastWithLocation() {
        if (ActivityCompat.checkSelfPermission(
                this,
                Manifest.permission.ACCESS_FINE_LOCATION
            ) != PackageManager.PERMISSION_GRANTED && ActivityCompat.checkSelfPermission(
                this,
                Manifest.permission.ACCESS_COARSE_LOCATION
            ) != PackageManager.PERMISSION_GRANTED
        ) {
            // TODO: Consider calling
            //    ActivityCompat#requestPermissions
            // here to request the missing permissions, and then overriding
            //   public void onRequestPermissionsResult(int requestCode, String[] permissions,
            //                                          int[] grantResults)
            // to handle the case where the user grants the permission. See the documentation
            // for ActivityCompat#requestPermissions for more details.
            return
        }
        fusedLocationClient.lastLocation.addOnSuccessListener { location ->
            if (location != null) {
                val locationData = "Latitude: ${location.latitude}, Longitude: ${location.longitude}"
                Toast.makeText(this@LocationForegroundService, locationData, Toast.LENGTH_SHORT).show()
            }
        }.addOnFailureListener { exception ->
            // Handle location retrieval failure
            Toast.makeText(this@LocationForegroundService, "Location retrieval failed", Toast.LENGTH_SHORT).show()
        }
    }


    fun isServiceRunning(): Boolean {
        return isServiceRunning
    }

    inner class LocationServiceBinder : Binder() {
        fun getService(): LocationForegroundService {
            return this@LocationForegroundService
        }
    }
}


# i don't have NOTIFICATION_CHANNEL_ID if you add id than automatically forground working started //:
